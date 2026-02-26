Plan: Simplified QA Intelligence Platform
TL;DR: Build 6 focused Node.js microservices in order of dependency. Start with API Gateway and Data Layer, then add Automation & Metrics engines, Orchestration, and CI/CD Ingestion. Each service has clear inputs/outputs and small testable functions. Keep logic simple: no advanced patterns, straightforward REST calls, basic database queries. Reference the Man2Auto architecture for domain concepts (coverage, risk scoring, confidence) but avoid over-engineering.

PHASE 1: API DESIGN
1.1 API Gateway Service
Purpose: Single entry point for all requests. Routes to backend services. No complex auth initially.

Endpoints to build (in order of implementation):

1. POST /api/v1/automation/generate
   Input: { testDocs, swaggerSpec, storyId, strategy } 
   Output: { requestId, status: "pending" }

2. GET /api/v1/automation/status/:requestId
   Input: requestId (path param)
   Output: { requestId, status, scripts[], metrics }

3. GET /api/v1/metrics/coverage/:storyId
   Input: storyId (path param)
   Output: { storyId, coveragePercent, testedStories, totalStories }

4. POST /api/v1/scripts/:scriptId/approve
   Input: { approverId, decision: "approve|reject", notes }
   Output: { scriptId, status: "approved|rejected", auditId }

5. GET /api/v1/scripts/pending-approval
   Input: (none)
   Output: { pendingScripts[], riskScores[] }

6. POST /api/v1/webhooks/ci
   Input: { buildId, testResults[], timestamp } (from GitHub/Jenkins)
   Output: { eventId, status: "queued" }

Key constraint: Each endpoint returns simple JSON. No complex response structures.

1.2 Automation Generator Service
Purpose: Generate test scripts using LLM + RAG. Classify results. Score confidence.

API endpoints (called by Orchestration Service):

POST /internal/generate
  Input: GenerationRequest { testDocs, apiSpec, storyId }
  Output: GenerationResponse { scripts[], classifications[], confidenceScores[] }

POST /internal/classify
  Input: { scriptText }
  Output: { type: "Positive|Negative|Boundary", confidence: 0.0-1.0 }

1.3 Automation Metrics Engine Service
Purpose: Calculate coverage, flakiness, risk scores.

API endpoints:

POST /internal/metrics/calculate
  Input: MetricsRequest { storyId, moduleId, timeRange }
  Output: MetricsResponse { 
    coverage%, 
    flakyIndex%, 
    riskScore, 
    automationDebtIndex 
  }

GET /internal/metrics/flaky/:testId
  Input: testId (path param)
  Output: { testId, failurePercent, trend }


  1.4 Orchestration Service
Purpose: Coordinate workflow. Route to approval based on risk.

API endpoints:

POST /internal/workflow/execute
  Input: GenerationRequest
  Output: OrchestrationResult { 
    requestId, 
    scripts[], 
    approvalPath: "auto|manual",
    riskLevel: "LOW|MEDIUM|HIGH"
  }

GET /internal/workflow/status/:requestId
  Input: requestId
  Output: { requestId, status: "draft|pending-approval|approved|rejected" }


  1.5 CI/CD Ingestion Service
Purpose: Receive CI webhooks. Store execution data. Publish events.

API endpoints:

POST /webhooks/github/push
  Input: { repository, branch, tests[], results[] }
  Output: { eventId, status: "received" }

POST /webhooks/jenkins/job-complete  
  Input: { jobId, buildNumber, testResults[] }
  Output: { eventId, status: "received" }



  1.6 Data Layer Service
Purpose: Abstract database access. Provide simple query interface.

API endpoints (internal only):

POST /internal/db/query
  Input: { table, operation: "select|insert|update", data }
  Output: { rows[], id (for inserts) }

POST /internal/db/vector-search
  Input: { query, topK: 5, entityType }
  Output: { results[{ id, similarity, content }] }



  PHASE 2: DATABASE DESIGN
2.1 Relational Database (PostgreSQL)
Tables required (in order of creation):

1. modules
   - id (PK)
   - name, component
   - risk_score (float), coverage_percent
   - created_at, updated_at

2. stories  
   - id (PK)
   - title, description
   - module_id (FK), coverage_percent
   - status: "not-started|in-progress|completed"

3. test_cases
   - id (PK)
   - title, description
   - module_id (FK), story_id (FK)
   - priority: "LOW|MEDIUM|HIGH"
   - vector_embedding_id (FK to vector DB)

4. automation_scripts
   - id (PK)
   - script_text, type: "Positive|Negative|Boundary"
   - story_id (FK), confidence_score (0.0-1.0)
   - status: "draft|pending-approval|approved|rejected"
   - created_at, approved_by, approved_at

5. ci_executions
   - id (PK)
   - build_id, test_case_id (FK)
   - status: "pass|fail|flaky", duration_ms
   - timestamp, branch, commit_sha

6. defects
   - id (PK)
   - story_id (FK), severity: "LOW|MEDIUM|HIGH"
   - frequency (int), last_seen

7. metrics_history
   - id (PK)
   - story_id or module_id (FK)
   - coverage_percent, flaky_index, risk_score, debt_index
   - recorded_at (timestamp)

8. approval_requests
   - id (PK)
   - script_id (FK), risk_level: "LOW|MEDIUM|HIGH"
   - requested_at, requested_by
   - approver_id, decision: "approved|rejected|pending"
   - decided_at, notes

9. audit_logs
   - id (PK)
   - action: "generate|approve|reject|execute"
   - actor_id, entity_id, entity_type
   - details (JSON), timestamp



   Key constraint: No complex relationships. Each table focusses on one entity.

2.2 Vector Database (pgvector)
Table:

embeddings
  - id (PK)
  - entity_id, entity_type: "test_case|api_doc|strategy_doc"
  - embedding (vector), metadata JSON
  - created_at



  Simplification: Only store embeddings for test cases. Don't over-embed.

2.3 Querying Strategy
Simple queries only:

Select by ID, FK
Order by timestamp
Count/aggregate for metrics
Cosine similarity search on vectors (pgvector built-in)
No complex joins initially. Keep data normalized but simple.

PHASE 3: WEB LAYER
3.1 Backend REST API (API Gateway)
Express.js server, port 3000
Middleware: request validation, error handler, logging
Route handlers for the 6 endpoints listed in Phase 1
3.2 Simple Web Dashboard (React/Vue.js)
Pages needed:

1. /dashboard
   - Cards: Total stories, Coverage %, Flaky tests count
   - Simple top-level metrics

2. /stories/:storyId
   - Story title, module, coverage %
   - List of test cases (checkbox: automated / not automated)
   - Coverage progress bar

3. /approvals
   - Table of pending approval requests
   - Script preview, risk level, approve/reject buttons
   - Simple approval form

4. /metrics
   - Chart: coverage trend per release
   - Table: modules, coverage %, risk scores
   - Filter by story/module

5. /audit-logs
   - Table: action, actor, timestamp, details
   - Filter by action type, date range



   Tech stack:

Frontend: React 18 + Axios (no Redux, no complex state management)
Build: Vite
Styling: Tailwind CSS (utility-first, minimal custom CSS)
Key constraint: No complex UI. Table + cards + charts. Use open-source components (recharts for graphs).

PHASE 4: DESIGN GUIDELINES
4.1 Simplicity Principles
One job per function: Each function does ONE thing well
No inheritance chains: Use composition and simple interfaces
No magic: No decorators, middleware chains, or AOP initially
Obvious naming: Class/function names clearly state purpose
Example:

// ✅ GOOD: Simple, clear
class CoverageCalculator {
  calculateByStory(storyId: string): number { ... }
}

// ❌ AVOID: Over-engineered
class AbstractMetricsProcessor 
  implements ICalculable, IComposable, IObservable { ... }



4.2 Testable Pieces Strategy
Each service = one Express app
Separate logic from routes:
routes/automation.ts → calls → services/AutomationGeneratorService
services/ → calls → repositories/ → database
Mock dependencies at service layer, not framework layer
Each function ≤ 20 lines of code (break on complexity)
Test structure:

tests/
  ├── unit/
  │   ├── AutomationGeneratorService.test.ts
  │   ├── CoverageCalculator.test.ts
  │   └── ConfidenceScorer.test.ts
  └── integration/
      ├── api.gateway.test.ts
      └── orchestration.workflow.test.ts
 
4.3 Data Flow (Keep Linear)
API Gateway (request) 
  → Orchestration Service (delegates)
    → Automation Generator (generates scripts)
      → Data Layer (query vector DB for context)
  → Metrics Engine (calculates metrics)
      → Data Layer (query relational DB)
  → Orchestration (combines results, routes approval)
  → API Gateway (returns response)

No circular dependencies. No event loops initially (message broker is async, decoupled).

4.4 Error Handling (3 layers)

Layer 1 (API Gateway): Catch all errors → 500 response
Layer 2 (Services): Log & propagate
Layer 3 (Repository): Log & throw

All errors logged with: { error, service, function, context, timestamp }

4.5 Configuration
Environment variables only (.env file)
No hardcoded values
Services auto-configure from ENV
Example:

DB_HOST=localhost
DB_PORT=5432
LLM_API_KEY=sk-...
VECTOR_DB_CONNECTION=postgresql://user:pass@host:5432/vectordb



4.6 Versioning
API endpoints versioned: /api/v1/...
Database migrations named: 001_init_tables.sql, 002_add_metrics.sql
Service versions in package.json only
4.7 Monitoring & Logging
Structured logs (JSON format)
Fields: timestamp, service, level, message, context
No console.log() — use Winston or Pino logger
Log levels: ERROR, WARN, INFO (not DEBUG initially)
4.8 Code Organization (Each Service)


src/
  ├── routes/          (Express route handlers)
  ├── services/        (Business logic)
  ├── repositories/    (Data access)
  ├── models/          (TypeScript interfaces/types)
  ├── middleware/      (Auth, validation, error)
  ├── utils/           (Helpers, constants)
  ├── config/          (Environment, database)
  └── app.ts           (Express app initialization)

tests/
  ├── unit/
  └── integration/


4.9 Git & CI/CD
Branch naming: feature/, bugfix/, hotfix/
Commit messages: TYPE(service): description (e.g., feat(api-gateway): add approval endpoint)
PR review before merge
GitHub Actions: Lint → Test → Deploy on main branch
4.10 Deployment (Start Simple)
Docker Compose for local dev
Single Docker image per service
Shared .env for all services during dev
PostgreSQL + pgvector in Docker containers
API Gateway exposed on localhost:3000

IMPLEMENTATION ORDER
Data Layer Service (foundation)

Database schema
Connection pooling
Simple query interface
API Gateway (request entry)

Express setup
Route handlers (return mocks initially)
Error middleware
Automation Generator Service (core logic)

RAG retriever (queries vector DB)
Classifier (categorize scripts)
Confidence scorer
Automation Metrics Engine (calculations)

Coverage calculator
Flaky index calculator
Risk score calculator
Orchestration Service (workflow)

Workflow engine (state machine)
Approval router (risk-based)
Audit logger
CI/CD Ingestion Service (events)

Webhook receiver
Event validator and enricher
Message broker publisher
Web Dashboard (UI)

Dashboard page
Story detail page
Approval queue page
Metrics page


KEY DECISIONS
Decision	     Choice	          Why
Microservices   count	Keep 6 (but simplify each)	Matches architecture, clear separation of concerns
Communication	REST + Message Broker (hybrid)	REST for sync requests, broker for async events
Database	PostgreSQL + pgvector	Single relational DB, vector extension, no external vector DB setup
Auth	JWT tokens (basic)	Simple, stateless, sufficient for internal services
LLM integration	OpenAI API (or local alternative)	Deferred; assume external LLM available
Message broker	RabbitMQ (or Kafka)	Optional for CI/CD events initially; can add later
Frontend framework	React + Tailwind	Simple, widely known, minimal dependencies
Node.js version	18 LTS	Stable, supported, TypeScript ready


SUCCESS CRITERIA
 API Gateway routes requests to all 6 services
 Data Layer CRUD operations tested with real PostgreSQL
 Automation Generator produces scripts with confidence scores
 Metrics Engine calculates coverage & risk accurately
 Orchestration routes high-risk modules to approval queue
 CI/CD webhooks ingest and store execution data
 Dashboard displays metrics and approval queue
 Full workflow tested end-to-end (API → generate → approve → dashboard)
 All services deployable via Docker Compose

