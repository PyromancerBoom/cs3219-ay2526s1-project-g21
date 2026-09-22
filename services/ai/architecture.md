
  AI Service Technical Summary

  What It Does

  The AI service is a Go-based intelligent code assistance microservice that provides contextual, LLM-powered help
   for coding interview practice. It offers four core capabilities:

  1. Code Explanations - Multi-level explanations (beginner/intermediate/advanced) of user code
  2. Progressive Hints - Graduated hints for problems without revealing solutions
  3. Test Generation - Automated unit test creation for submitted solutions
  4. Refactoring Tips - Code quality suggestions and best practices

  Beyond basic generation, it includes production ML infrastructure: feedback collection, automated model
  fine-tuning pipeline, A/B testing for model comparison, and performance monitoring.

  Current LLM: Google Gemini (via official GenAI SDK), but architected for multi-provider support.

  ---
  Why It Exists as a Separate Microservice

  Engineering Motivations:

  1. Specialized Resource Requirements
    - LLM API calls have high latency (1-5s) and rate limits
    - Needs independent scaling from main application
    - Resource-intensive ML operations (fine-tuning, export jobs) shouldn't impact core services
  2. Distinct Domain Logic
    - Complex prompt engineering and template management
    - ML-specific concerns: A/B testing, feedback loops, model versioning
    - Provider-specific error handling (rate limits, quota errors)
  3. Iterative Development Velocity
    - Prompt tuning requires rapid iteration without touching main codebase
    - Model experimentation benefits from isolated deployment
    - Breaking changes to AI responses don't cascade to other services
  4. Cost Management & Observability
    - Centralized LLM cost tracking
    - Dedicated monitoring for AI-specific metrics (latency, feedback scores, model performance)
    - Easy to add caching layer or swap providers without changes elsewhere
  5. Security Isolation
    - LLM API keys isolated from main application
    - User code sent to external APIs contained in single service
    - Easier to audit/review AI-specific data handling

  Constraint: If embedded in monolith, high-latency AI operations would block core features, and ML
  experimentation would require coordinated deployments.

  ---
  Architecture Overview

  Core Stack

  - Language: Go 1.23
  - Router: Chi (lightweight, idiomatic)
  - Database: PostgreSQL + GORM
  - LLM Client: Google GenAI SDK
  - Logging: Uber Zap (structured)
  - Scheduling: Robfig cron

  High-Level Component Diagram

  ┌─────────────────────────────────────────────────────────────┐
  │                        HTTP API Layer                        │
  │  Chi Router + Middleware (Validation, CORS, Logging, Timeout)│
  └──────────────────┬──────────────────────────────────────────┘
                     │
          ┌──────────┼──────────┐
          │          │           │
          ▼          ▼           ▼
      ┌────────┐ ┌─────────┐ ┌──────────┐
      │AI      │ │Feedback │ │Model Mgmt│
      │Handler │ │Handler  │ │Handler   │
      └────┬───┘ └────┬────┘ └────┬─────┘
           │          │           │
           │          │           │
           ▼          ▼           ▼
      ┌──────────────────────────────────┐
      │      Business Logic Layer         │
      │                                   │
      │  ┌──────────────┐  ┌───────────┐│
      │  │Prompt Manager│  │LLM Provider││
      │  │(Templates)   │  │  Registry  ││
      │  └──────────────┘  └─────┬─────┘│
      │                           │      │
      │  ┌───────────────┐  ┌────▼────┐ │
      │  │Feedback       │  │ Gemini  │ │
      │  │Manager        │  │ Client  │ │
      │  │+ Context Cache│  │ (A/B)   │ │
      │  └───────────────┘  └─────────┘ │
      └──────────┬──────────────┬────────┘
                 │              │
                 │              │
      ┌──────────▼──────────────▼──────────┐
      │       Infrastructure Layer          │
      │                                     │
      │  PostgreSQL      Google Gemini API │
      │  (Feedback DB)   (Generation)      │
      │                                     │
      │  Vertex AI       Cron Scheduler    │
      │  (Fine-tuning)   (Export/Tuning)   │
      └─────────────────────────────────────┘

  Data Flow (Request → Response)

  HTTP POST /ai/explain
    ↓
  1. Middleware Chain
     - Deserialize & validate request
     - Normalize language/detail_level
     - Store in context
    ↓
  2. AI Handler
     - Extract validated request
     - Generate/use request_id
    ↓
  3. Prompt Manager
     - Load YAML template (embedded)
     - Execute Go template with user data
     - Return: base_prompt + variant_prompt
    ↓
  4. LLM Provider (Gemini Client)
     - A/B Test: Select model via weighted random
     - Call Google GenAI API
     - Handle errors (rate limit → 429, quota → 503)
    ↓
  5. Response Processing
     - Store context in cache (15min TTL)
     - Return JSON: {explanation, request_id, model_version, processing_time}

  Feedback Loop Flow

  User clicks thumbs-up/down
    ↓
  POST /feedback/{request_id}
    ↓
  1. Feedback Handler
     - Lookup context from cache
     - Persist to PostgreSQL (prompt + response + is_positive)
     - Remove from cache
    ↓
  2. Scheduled Job (cron: e.g., 2 AM daily)
     - Query unexported feedback
     - Filter positive only
     - Export to JSONL (Gemini training format)
    ↓
  3. Automated Fine-tuning (if enabled & threshold met)
     - Upload JSONL to GCS
     - Create Vertex AI tuning job
     - Poll until complete
     - Activate model with 10% traffic
    ↓
  4. A/B Testing
     - New model serves 10% of requests
     - Compare feedback stats vs base model
     - Adjust traffic weight based on performance

  ---
  Core Patterns & Abstractions

  1. Provider Registry Pattern (Pluggable LLMs)

  Files: internal/llm/{provider.go, registry.go, gemini/init.go}

  Pattern: Factory + Registry for multi-provider support

  // Core abstraction
  type Provider interface {
      GenerateContent(ctx, prompt, requestID, detailLevel) (*Response, error)
      GetProviderName() string
  }

  // Self-registration in gemini/init.go
  func init() {
      llm.RegisterProvider("gemini", NewGeminiProvider)
  }

  Why: Adding OpenAI/Claude requires only implementing Provider interface + init() registration. No changes to
  handlers.

  Trade-off: Global registry simplifies addition but uses global state (standard in Go plugin systems).

  ---
  2. Generic Validation Middleware

  File: internal/middleware/validation.go

  Pattern: Go 1.18+ generics for type-safe request handling

  type Validator interface { Validate() error }

  func ValidateRequest[T Validator]() func(http.Handler) http.Handler {
      // Generic middleware works with any request type
      // Validates, normalizes, stores in context
  }

  // Usage in handler
  req := GetValidatedRequest[*ExplainRequest](r)

  Why: Single middleware implementation for all endpoints. Eliminates repetitive validation code. Type-safe
  context retrieval.

  Trade-off: Requires Go 1.18+, slightly more complex middleware code.

  ---
  3. Template-Based Prompt System

  Files: internal/prompts/{manager.go, templates/*.yaml}

  Pattern: Template Method + Strategy with embedded resources

  # templates/explain.yaml
  base_prompt: |
    You are a helpful programming tutor...

  prompts:
    beginner: |
      Explain this {{.Language}} code in simple terms...
      Code: {{.Code}}

    intermediate: |
      Provide technical explanation with concepts...

  // manager.go
  //go:embed templates/*.yaml
  var templateFS embed.FS

  func (pm *PromptManager) BuildPrompt(mode, variant, data) string {
      template := pm.templates[mode][variant]
      return base_prompt + execute(template, data)
  }

  Why:
  - Easy prompt iteration without code changes
  - Consistent base instructions across all modes
  - Templates versioned with code (embedded)
  - Type-safe at compile time

  Trade-off: Must rebuild to change prompts (acceptable for production stability).

  ---
  4. Cache-Aside Pattern for Feedback Context

  File: internal/feedback/context_cache.go

  Pattern: In-memory TTL cache + background cleanup

  type ContextCache struct {
      data      map[string]*CacheEntry
      mu        sync.RWMutex
      ttl       time.Duration
      cleanup   time.Duration
  }

  // Background goroutine cleans expired entries every 5 min

  Why:
  - No DB lookup on AI requests (latency-sensitive)
  - Feedback typically submitted within minutes
  - Automatic memory leak prevention

  Trade-off: Feedback lost if service restarts (acceptable - feedback is voluntary).

  ---
  5. Weighted Random Selection for A/B Testing

  File: internal/llm/gemini/client.go

  Pattern: Simple weighted random model selection

  func (c *Client) selectModel() (string, string) {
      activeModels := queryActiveModelsFromDB()
      baseWeight := 100 - sum(activeModels.TrafficWeights)

      random := rand.Intn(100)
      if random < baseWeight {
          return BASE_MODEL, "base"
      }

      // Iterate fine-tuned models by cumulative weight
  }

  Configuration Example:
  - Base model: 70%
  - Fine-tuned v1: 20%
  - Fine-tuned v2: 10%

  Why: Real-time traffic adjustment, simple to debug, no external dependencies.

  Trade-off: No user-level consistency (same user may hit different models), no built-in statistical testing.

  ---
  6. Dependency Injection

  File: cmd/server/main.go

  Pattern: Constructor-based DI throughout

  // Bootstrap
  promptManager := prompts.NewPromptManager()
  provider := llm.NewProvider(cfg.Provider)
  feedbackMgr := feedback.NewFeedbackManager(db, cache)
  aiHandler := handlers.NewAIHandler(provider, promptManager, logger)

  Why: Explicit dependencies, testable via mocks, clear component boundaries.

  ---
  Notable Design Decisions & Trade-offs

  1. Go Over Python

  Decision: Use Go instead of typical Python ML service

  Rationale:
  - Low-latency HTTP handling (Chi router)
  - Better concurrency primitives (goroutines for cache cleanup, job scheduling)
  - Single binary deployment (no virtualenv/dependencies)
  - Type safety reduces prompt/data bugs
  - Strong standard library (templating, HTTP, JSON)

  Trade-off: Smaller ML ecosystem (but not needed - just calling LLM APIs).

  ---
  2. In-Memory Cache vs. Database for Context

  Decision: Use TTL cache instead of persisting all request context

  Rationale:
  - Feedback submitted within minutes (15min window sufficient)
  - Reduces DB load on high-frequency AI requests
  - Automatic cleanup prevents memory growth

  Trade-off: Context lost on restart (acceptable - feedback is optional).

  ---
  3. Embedded YAML Templates

  Decision: //go:embed templates in binary vs. loading from filesystem

  Rationale:
  - Single binary deployment (no config files to manage)
  - No runtime file loading errors
  - Templates versioned with code
  - Faster startup (no disk I/O)

  Trade-off: Must rebuild to change prompts (enforces version control discipline).

  ---
  4. Vertex AI REST API vs. SDK

  Decision: Raw HTTP requests with gcloud auth instead of official Vertex AI Go SDK

  File: internal/tuning/gemini_tuner.go

  Rationale:
  - Official SDK may have limited fine-tuning API support
  - More control over request/response handling
  - gcloud CLI simplifies credential management in GCP environments

  Trade-off: Manual request construction, must track API changes manually.

  ---
  5. PostgreSQL vs. NoSQL

  Decision: Use PostgreSQL for feedback storage

  Rationale:
  - ACID transactions for feedback consistency
  - Relational data (model_versions ← ai_feedbacks)
  - Excellent GORM integration
  - Well-suited for analytics queries (feedback stats by model)

  Trade-off: More setup than file storage, requires migration management.

  ---
  6. Provider Registry with Global State

  Decision: Global registry vs. dependency injection for providers

  Rationale:
  - Standard Go idiom for plugin systems
  - Clean provider addition (just init() registration)
  - No central list to maintain

  Trade-off: Global state (generally discouraged) makes testing slightly harder.

  ---
  Key Components in Detail

  Prompt Manager (internal/prompts/manager.go)

  Responsibilities:
  - Load YAML templates at startup (embedded via go:embed)
  - Validate templates compile successfully
  - Execute templates with request data (language, code, hints, etc.)
  - Return complete prompts (base + variant)

  Template Data Structure:
  type PromptData struct {
      Language         string
      Code             string
      QuestionTitle    string
      QuestionDesc     string
      PreviousHints    []string
      HintsRequested   int
  }

  ---
  Gemini Client (internal/llm/gemini/client.go)

  Responsibilities:
  - Authenticate with Google GenAI SDK
  - Implement A/B testing model selection
  - Call LLM API with prompt
  - Handle provider-specific errors (rate limit, quota, API key)
  - Return standardized response with metadata

  Error Handling:
  - Rate limit → HTTP 429 with retry-after
  - Quota exceeded → HTTP 503
  - Invalid API key → HTTP 500
  - Timeout → HTTP 504

  ---
  Feedback Manager (internal/feedback/feedback_manager.go)

  Responsibilities:
  - Accept user feedback (thumbs up/down)
  - Retrieve context from cache
  - Persist to PostgreSQL with prompt/response
  - Export feedback to JSONL format
  - Calculate feedback statistics by model

  Database Schema:
  type AIFeedback struct {
      ID             uint
      RequestID      string
      Prompt         string
      Response       string
      IsPositive     bool
      ModelVersion   string
      CreatedAt      time.Time
      IsExported     bool
      ExportedAt     *time.Time
  }

  ---
  Feedback Exporter Job (internal/jobs/feedback_exporter.go)

  Responsibilities:
  - Run on cron schedule (e.g., 2 AM daily)
  - Query unexported positive feedback
  - Export to JSONL format:
  {"contents": [{"role": "user", "parts": [{"text": "..."}]}, {"role": "model", "parts": [{"text": "..."}]}]}
  - Mark as exported in database
  - Optionally trigger automated fine-tuning

  ---
  Gemini Tuner (internal/tuning/gemini_tuner.go)

  Responsibilities:
  - Upload JSONL to Google Cloud Storage
  - Create Vertex AI supervised tuning job
  - Poll job status until complete (30s intervals)
  - Update model_versions table with endpoint
  - Activate model with initial traffic weight (10%)

  Configuration:
  - Base model, learning rate, epoch count, adapter size
  - GCP project, region, GCS bucket

  ---
  API Endpoints Summary

  AI Generation

  - POST /api/v1/ai/explain - Code explanations
  - POST /api/v1/ai/hint - Progressive hints
  - POST /api/v1/ai/tests - Test generation
  - POST /api/v1/ai/refactor-tips - Refactoring suggestions

  Feedback

  - POST /api/v1/ai/feedback/{request_id} - Submit feedback
  - GET /api/v1/ai/feedback/export - Export JSONL
  - GET /api/v1/ai/feedback/stats - Feedback statistics

  Model Management

  - GET /api/v1/ai/models - List models
  - GET /api/v1/ai/models/{id}/stats - Model performance
  - PUT /api/v1/ai/models/{id}/traffic - Update traffic weight
  - PUT /api/v1/ai/models/{id}/deactivate - Deactivate model

  Health

  - GET /healthz - Liveness
  - GET /readyz - Readiness

  ---
  Integration Points

  1. Google Gemini API - Generation (via google.golang.org/genai)
  2. Vertex AI REST API - Fine-tuning (gcloud auth)
  3. PostgreSQL - Feedback persistence (GORM)
  4. Frontend - CORS configured for localhost:5173 + production CloudFront
  5. Other Microservices - None directly (standalone with HTTP API)

  ---
  Summary for Senior Engineer

  This is a production-grade ML microservice demonstrating:

  1. Clean Architecture - Layered design (handlers → business logic → infrastructure)
  2. Extensibility - Plugin pattern for LLM providers, template-based prompts
  3. Type Safety - Go generics for validation, strong typing throughout
  4. ML Ops - Complete feedback loop: collection → export → fine-tuning → deployment → A/B testing
  5. Resilience - Error handling, rate limiting, timeouts, graceful shutdown
  6. Performance - In-memory caching, efficient DB queries, concurrent processing

  Key Insight: The service isn't just an LLM wrapper—it's a complete ML lifecycle platform with automated model
  improvement, experimental infrastructure (A/B testing), and production-grade observability. The architecture
  anticipates evolution (multi-provider support, prompt iteration) while maintaining type safety and clear
  separation of concerns.

