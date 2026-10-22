# Open CS Labs --- Architecture and Deployment Blueprint

**Status:** Initial proposed architecture; technology decisions marked
*proposed* must be validated before being treated as final.\
**Product:** Open CS Labs --- course learning, SQL/DSA practice,
autograding, discussion, RAG-based doubt support, and professor
analytics.\
**Primary constraint:** A student team of 16, split into 4 teams, must
be able to build, test, deploy, and hand over the system.

## 1. Product scope

### Student capabilities

-   Register, log in, and manage profile.
-   View enrolled courses, announcements, theory/lecture material,
    assignments, and deadlines.
-   Practice SQL and DSA in a Monaco-based editor.
-   Submit solutions, view verdicts, and review submission history.
-   Save drafts locally and optionally synchronize drafts to the server.
-   Ask course-related questions using permission-aware RAG.
-   Participate in authorized course chatrooms.
-   View personal progress and weekly activity reports.

### Professor capabilities

-   Manage authorized courses and enrollments.
-   Publish announcements and learning materials.
-   Create/manage problems, assignments, due dates, and test cases.
-   Review submissions and progress.
-   View grading statistics and configurable relative-grade
    recommendations.
-   Approve final grades and manual overrides with an audit trail.
-   Receive weekly student/course reports.

### Explicit non-goals for the first release

-   Recreating all of LeetCode.
-   Scraping undocumented or private LeetCode endpoints.
-   Running untrusted code inside the main backend.
-   Starting with many independent microservices.
-   Claiming production-scale capacity before load testing.
-   Automatically finalizing grades without professor approval.

## 2. Proposed technology stack

This is a starting proposal, not a claim that all choices are already
benchmark-proven.

  -----------------------------------------------------------------------
  Area                    Proposed choice         Rationale / validation
  ----------------------- ----------------------- -----------------------
  Frontend                React + TypeScript +    Familiar ecosystem,
                          Vite                    fast local development,
                                                  static build

  UI                      Tailwind CSS +          Consistent reusable
                          shadcn/ui               components

  Routing/data            React Router + TanStack Route structure and
                          Query                   server-state handling

  Code editor             Monaco Editor           VS Code-style editing
                                                  experience

  Backend                 Python + FastAPI        Typed HTTP APIs,
                                                  OpenAPI, Python
                                                  ecosystem for RAG

  Validation              Pydantic                Request/response
                                                  validation

  ORM/migrations          SQLAlchemy 2 + Alembic  Explicit data access
                                                  and versioned
                                                  migrations

  Main database           PostgreSQL              Relational transactions
                                                  and constraints

  Vector search           pgvector extension      Keep initial RAG
                                                  storage close to
                                                  course/document
                                                  metadata

  Chat store              Cassandra,              Dedicated room-message
                          **conditional on        history if justified
                          benchmark and           
                          operational review**    

  Cache/job broker        Redis, optional         Cache, rate limits,
                                                  queue broker or
                                                  coordination

  Background jobs         Celery + Redis, if      Reports and scheduled
                          adopted                 tasks

  Code judge              Judge0 or another       Never execute untrusted
                          reviewed isolated judge code in the API process

  SQL practice            Isolated disposable     No access to production
                          PostgreSQL environment  data or credentials

  Files                   S3-compatible object    Slides, generated PDFs,
                          storage or managed      report artifacts
                          equivalent              

  Testing                 pytest, HTTPX, Vitest,  Unit, integration, and
                          React Testing Library,  end-to-end coverage
                          Playwright              

  Load testing            k6 and/or Locust        Reproducible load
                                                  profiles and
                                                  measurements

  Local environment       Docker Compose          Repeatable dependencies

  CI                      GitHub Actions or       Tests/build checks
                          equivalent              before merge
  -----------------------------------------------------------------------

### Decision gates

-   **Cassandra:** do not assume it is faster end-to-end. Benchmark
    equivalent PostgreSQL and Cassandra workloads on the same hardware
    and dataset. Include operational complexity, backup/restore,
    consistency, and team familiarity in the decision. If Cassandra is
    retained, PostgreSQL remains the source of truth for users, room
    membership, courses, and permissions.
-   **Redis:** add only when a specific need exists (queue broker, rate
    limiting, cache, or multi-instance event fan-out). It is not the
    authoritative source of business records.
-   **Judge0:** validate deployment/security requirements and resource
    limits before accepting student code. Do not expose the judge
    directly without an authenticated, rate-limited backend integration.
-   **LLM provider:** choose after comparing quality, latency, privacy,
    cost, and data-retention terms. Keep provider access behind a
    backend adapter.
-   **LeetCode:** rely on supported, permitted integration only. The
    platform must work with its own problems and autograder.

## 3. High-level architecture

``` mermaid
flowchart TB
    Browser[React + TypeScript]
    API[FastAPI modular monolith]
    PG[(PostgreSQL)]
    PV[(pgvector extension)]
    CAS[(Cassandra - conditional)]
    REDIS[(Redis - optional)]
    WORKER[Background worker]
    JUDGE[Isolated code judge]
    SQLBOX[Disposable SQL sandbox]
    OBJ[Object storage]
    LLM[LLM / embedding provider]

    Browser -->|HTTPS REST| API
    Browser <-->|WSS| API
    API --> PG
    API --> PV
    API -->|Chat history only| CAS
    API --> REDIS
    API -->|Enqueue jobs| REDIS
    REDIS --> WORKER
    WORKER --> PG
    WORKER --> OBJ
    API -->|DSA submission| JUDGE
    API -->|SQL run| SQLBOX
    API --> LLM
```

The diagram shows logical boundaries, not a requirement to run every
component on a separate server. In development, services can run in
Docker Compose. Production topology must be chosen based on load,
security, and available infrastructure.

## 4. Backend module boundaries

Keep one deployable backend initially, with modules and ownership
clearly separated.

1.  **Identity and access:** users, sessions, roles, course membership,
    authorization policies.
2.  **Courses and content:** courses, enrollments, announcements,
    lectures, readings, file metadata.
3.  **Problems and assignments:** problem statements, public examples,
    assignments, due dates, test-case metadata.
4.  **Submission orchestration:** submission records, queue state, judge
    adapter, verdict normalization, retry/idempotency handling.
5.  **SQL practice:** isolated database provisioning, query execution
    limits, cleanup.
6.  **Chat:** room membership checks, WebSocket connection handling,
    persistence adapter, history pagination.
7.  **RAG:** document ingestion, chunking, embedding, permission-aware
    retrieval, answer generation, feedback.
8.  **Grading and analytics:** score aggregation, grading policies,
    review/approval, audit history.
9.  **Reports:** scheduled jobs, PDF generation, secure artifact access,
    delivery state and retry.
10. **Operations:** health checks, metrics, structured logs,
    configuration validation.

Modules may call each other through explicit service interfaces. Avoid
circular imports and direct cross-module table manipulation where a
module owns the data.

## 5. Data ownership

### PostgreSQL --- authoritative application data

Proposed logical tables (names can change through reviewed migrations):

-   Identity: `users`, `sessions` or session store, `roles`/role
    mapping.
-   Courses: `courses`, `enrollments`, `course_memberships`.
-   Content: `announcements`, `learning_resources`, `file_assets`.
-   Practice: `problems`, `problem_examples`, `assignments`,
    `assignment_problems`.
-   Submissions: `submissions`, `submission_results`, `drafts`,
    `assignment_scores`.
-   Grading: `grading_policies`, `grading_runs`,
    `grade_recommendations`, `final_grades`, `grade_audit_events`.
-   RAG: `knowledge_documents`, `document_chunks`, embedding
    columns/indexes via pgvector, `rag_feedback`.
-   Reports: `report_jobs`, `student_reports`, `report_deliveries`.
-   Audit: security-sensitive or grading-related audit events.

Use foreign keys and constraints where appropriate. Do not store hidden
test case content in responses sent to students. Consider separate
storage/permissions for hidden tests.

### Cassandra --- conditional chat history store

If benchmarks and operations review justify Cassandra, it owns
persistent message history only. Room identity, membership, course
permissions, and user profile data remain authoritative in PostgreSQL.

Example query to support: "fetch the newest messages for room R within a
bounded time bucket." A candidate schema might use a partition key based
on `(room_id, bucket_date)` and a time-sortable message ID. Final schema
must be based on actual access patterns, expected partition sizes,
retention, pagination, and ordering requirements.

Do not assume: - relational joins are available; - arbitrary filters are
efficient; - a successful database write means all WebSocket recipients
received the message; - message order is globally consistent across all
partitions.

Define message ID generation, idempotency key, retry behavior, timestamp
semantics, history pagination, and reconnect/backfill behavior.

### Redis --- ephemeral coordination only

If adopted, use it for explicit purposes such as queue brokering, rate
limits, short-lived cache, or cross-instance pub/sub. Redis pub/sub
alone is not a durable message history. Do not store the only copy of
grades, submissions, or chat messages in Redis.

### Object storage

Store slide files, generated PDFs, and other binary assets in object
storage. PostgreSQL stores metadata, ownership, access policy, checksum,
and storage key. Use short-lived signed URLs or a permission-checking
download endpoint. Never make private reports publicly readable.

## 6. Core request flows

### A. Authentication and course access

1.  Browser sends credentials over HTTPS.
2.  Backend validates credentials and creates a secure session/token.
3.  Protected endpoints resolve the current user on the server.
4.  Course and resource access is checked on every request.
5.  UI visibility is not treated as authorization.

### B. DSA submission

1.  Backend validates problem, language, user, and request limits.
2.  Persist submission as `QUEUED` and return its ID.
3.  Send a durable job to the queue or judge integration; make retries
    idempotent.
4.  Execute in an isolated sandbox with CPU, memory, wall-clock,
    process, filesystem, and network restrictions.
5.  Normalize results such as `ACCEPTED`, `WRONG_ANSWER`,
    `TIME_LIMIT_EXCEEDED`, `RUNTIME_ERROR`, `COMPILE_ERROR`, and
    infrastructure failure.
6.  Persist result and safe public feedback; never return hidden tests
    or secrets.
7.  Frontend polls initially; add event/WebSocket updates only when
    implemented and tested.

Infrastructure failures must be distinguishable from a student's wrong
answer.

### C. SQL practice

1.  Backend validates the exercise and query size/time limits.
2.  Execute only against an isolated disposable or resettable sandbox
    dataset.
3.  Use a restricted DB role with no access to production and no
    unnecessary privileges.
4.  Enforce statement timeout, memory/resource limits, row/result-size
    limits, and cleanup.
5.  Return bounded results and safe errors; log metrics without leaking
    sensitive data.

### D. Chat message

1.  Client establishes WSS and authenticates.
2.  Backend checks room membership and rate limits.
3.  Validate and size-limit the message.
4.  Persist the message to the selected store.
5.  Broadcast to connected authorized recipients after the chosen
    persistence policy succeeds.
6.  Use stable message IDs and client/server deduplication.
7.  On reconnect, fetch history after the last acknowledged message.
8.  For multiple backend instances, implement and test cross-instance
    fan-out; do not assume in-memory connections are shared.

### E. RAG doubt solving

1.  Ingest approved documents and record source/course/visibility
    metadata.
2.  Extract text, chunk it, and generate embeddings.
3.  Store chunks and embeddings with document and access metadata.
4.  Check user/course permissions before retrieval, not only after
    answer generation.
5.  Retrieve relevant chunks, pass them as untrusted context to the
    model, and request grounded answers with source references.
6.  Treat prompts in uploaded documents as untrusted; do not allow them
    to override system rules or grant access.
7.  Record feedback and safe operational metrics. Define retention and
    privacy policy before storing prompts.

### F. Weekly reports and relative grading

1.  A scheduled worker selects the reporting period and authorized
    course roster.
2.  Compute metrics from authoritative database records using a
    versioned definition.
3.  Generate PDF artifacts in background jobs and store them privately.
4.  Record job/report/delivery status and retry safely.
5.  Provide the professor a secure access path; never expose a report
    using guessable public URLs.
6.  Relative-grade recommendations are configurable and transparent.
    Show distributions and assumptions, handle small classes/zero
    variance/missing or excused work, record manual overrides, and
    require professor approval for final grades.

## 7. Security model

-   Enforce role-based access plus resource/course-level checks on the
    server.
-   Hash passwords with a maintained password-hashing library.
-   Use HTTPS in production; configure cookies, CSRF, CORS, and session
    expiry correctly.
-   Apply rate limits to authentication, chat, RAG, and submissions.
-   Keep secrets in environment variables or a secret manager; commit
    placeholders only.
-   Use parameterized queries and ORM-safe patterns.
-   Never execute arbitrary code or SQL with backend privileges.
-   Isolate judge and SQL sandbox networks from production data stores.
-   Do not expose hidden tests, private student reports, or cross-course
    documents.
-   Validate uploaded files and enforce size/type limits; scan or
    sandbox document processing where appropriate.
-   Redact credentials and sensitive data from logs.
-   Back up PostgreSQL and any durable Cassandra data if adopted; test
    restoration.
-   Define data retention, account deletion, report access, and audit
    policy.

## 8. Deployment plan

### Local development

-   Docker Compose for PostgreSQL and required local dependencies.
-   Cassandra/Redis only when included in the current development
    profile.
-   Separate frontend and backend processes or containers.
-   `.env.example` contains placeholder values; `.env` stays untracked.
-   Seed only synthetic development data.
-   Document a clean-clone setup and database migration command.

### CI

For each pull request: 1. Install from lockfiles. 2. Run
formatter/linter. 3. Run type checks. 4. Run unit tests. 5. Run
integration tests against disposable test services. 6. Build frontend
and backend images/artifacts. 7. Check migrations and security-sensitive
changes. 8. Fail the pipeline on required check failures.

### Staging

-   Use separate credentials and non-production data.
-   Run migrations, smoke tests, auth checks, and a limited load test.
-   Verify logs, health checks, backups, restore procedure, and judge
    isolation.

### Production

-   Use TLS and a supported deployment platform.
-   Store secrets in platform secret storage.
-   Use managed database services where feasible; if self-hosted,
    explicitly own patching, backups, monitoring, and recovery.
-   Review migration compatibility and rollback/recovery plan.
-   Set resource limits and alerts.
-   Run post-deploy smoke tests and monitor errors/latency.
-   Never claim deployment succeeded until the target environment is
    actually checked.

## 9. Observability and performance

Track at minimum: - API request count, error rate, and latency
p50/p95/p99. - WebSocket active connections, send failures, reconnects,
and end-to-end delivery latency. - Database query latency, connection
pool saturation, disk usage, and failed operations. - Submission queue
depth, queue wait time, execution time, verdict distribution, and
infrastructure failures. - RAG latency, retrieval results, provider
errors, and token/cost metrics where available. - Background job queue
age, retry count, PDF generation time, and delivery failures.

Never put secrets, full authorization headers, or unnecessary private
message contents in metrics/log labels.

## 10. Benchmark protocol

A benchmark result is meaningful only when configuration and workload
are reproducible.

Record: - machine/VM specifications and OS; - container and database
versions/configuration; - dataset size and distribution; -
request/message sizes and read/write mix; - concurrency, ramp-up,
warm-up, duration, and repetition count; - p50/p95/p99 latency,
throughput, errors, CPU, memory, disk, and network; - test scripts,
commit hash, date, and raw result files.

For PostgreSQL vs Cassandra chat comparison: - use the same generated
messages and room distribution; - run equivalent writes and room-history
reads; - include steady-state and increasing-concurrency runs; - measure
database latency separately from end-to-end WebSocket delivery; -
include failure recovery and operational cost in the decision; - never
fabricate or extrapolate results as if measured.

## 11. Team boundaries (16 people / 4 teams)

  -----------------------------------------------------------------------------
  Team                    Primary ownership             Must coordinate with
  ----------------------- ----------------------------- -----------------------
  1 --- Frontend          React app, shared UI, Monaco, API contracts, auth,
                          dashboards                    submission/chat states

  2 --- Core backend      Identity, courses, content,   All teams
                          PostgreSQL, API contracts     

  3 --- Judge/practice    DSA execution integration,    Backend, security,
                          SQL sandbox, submissions,     frontend
                          test cases                    

  4 --- Chat/RAG/platform Chat, Cassandra evaluation,   Backend, frontend, data
                          RAG,                          ownership
                          reports/infra/observability   
  -----------------------------------------------------------------------------

One person should own integration/release coordination. All teams work
in one repository with branch protection, small pull requests, reviewed
migrations, and a shared development environment. Do not let teams
independently invent incompatible API shapes or duplicate database
ownership.

## 12. Proposed repository layout

Adjust to existing repository if it already exists; do not blindly
overwrite files.

``` text
open-cs-labs/
├── apps/
│   ├── web/                    # React + TypeScript
│   └── api/                    # FastAPI modular monolith
├── packages/
│   └── api-client/             # Generated/shared client types if used
├── infra/
│   ├── docker/
│   ├── compose/                # Local profiles and service config
│   └── monitoring/             # Optional metrics configuration
├── tests/
│   ├── integration/
│   ├── e2e/
│   └── load/
├── docs/
│   └── ADRs/                   # Decision records (index in docs/ADRs/README.md)
├── API_CONTRACTS.md
├── ARCHITECTURE.md
├── DATABASE_SCHEMA.md
├── DEPLOYMENT.md
├── IMPLEMENTATION_PLAN.md
├── README.md
├── RULES.md
├── STATUS.md
├── TESTING.md
├── package.json
├── pnpm-workspace.yaml
├── .env.example
├── .gitignore
├── .nvmrc
└── .python-version
```

Top-level documentation lives at the **repository root**; `docs/` currently holds only
ADRs. This reflects the actual repository layout as verified in M0.

Do not create all directories just for appearance; add them when the
first implementation needs them.

## 13. Milestones

> **Canonical numbering:** `IMPLEMENTATION_PLAN.md` owns the authoritative milestone
> definitions (M0-M7). Where this section and that file differ, `IMPLEMENTATION_PLAN.md`
> wins. In particular, chat (M4) and RAG (M5) are separate milestones there, whereas an
> older draft of this section grouped them.

### M0 --- Decisions and repository foundation

-   Inspect current repository and team skills.
-   Confirm stack and benchmark decision gates.
-   Agree API conventions, module owners, branch/PR policy.
-   Create clean setup, lint/test/build CI, `.env.example`, and ADR
    template.

### M1 --- First vertical slice

-   Registration/login and server-side authorization.
-   Course list/enrollment and student dashboard shell.
-   PostgreSQL migrations and seed data.
-   Frontend connects to real API; no fake success responses.

### M2 --- Practice and submissions

-   Problem catalog, Monaco editor, draft saving.
-   Isolated DSA judge integration and safe SQL sandbox.
-   Submission lifecycle, verdicts, edge-case tests, rate limits.

### M3 --- Teaching workflows

-   Announcements, content, assignments, marks and professor views.
-   Access control and audit trails.

### M4 --- Chat and RAG

-   Implement chat protocol and history pagination.
-   Run PostgreSQL/Cassandra benchmark before committing to Cassandra
    operationally.
-   Add permission-aware document ingestion/retrieval and citations.

### M5 --- Reports, grading and deployment

-   Scheduled PDFs, secure delivery and retry tracking.
-   Configurable relative-grade analysis with professor approval.
-   Staging deployment, backup/restore drill, load tests, security
    review and handover docs.

## 14. Architecture Decision Records

Record significant choices under `docs/ADRs/`. Each ADR includes: -
Context/problem. - Requirements and constraints. - Options considered. -
Evidence/benchmark methodology and results, if applicable. - Decision
and trade-offs. - Security, operations, and migration implications. -
Revisit conditions.

Initial ADRs: 1. Frontend/backend stack. 2. PostgreSQL as authoritative
relational store. 3. Cassandra for chat: retain only if benchmark and
operations review justify it. 4. Judge isolation strategy. 5. RAG
storage/provider and privacy. 6. Deployment topology and backup/restore.

## 15. Release readiness checklist

-   [ ] Clean-clone setup succeeds from documented instructions.
-   [ ] No secrets or real student data in repository.
-   [ ] Authentication and resource-level authorization tests pass.
-   [ ] Database migrations apply on a clean environment.
-   [ ] Critical student/professor journeys pass end-to-end tests.
-   [ ] Student code and SQL are isolated from production data.
-   [ ] Hidden tests and private reports are not exposed.
-   [ ] WebSocket reconnect and history recovery are tested.
-   [ ] Background jobs retry safely and failures are visible.
-   [ ] Backups and restoration have been tested.
-   [ ] Health checks, logs, error metrics, and latency metrics are
    available.
-   [ ] Load/security test scope and limitations are documented.
-   [ ] Staging smoke tests pass.
-   [ ] Deployment and rollback/recovery instructions are current.
-   [ ] A maintainer can explain module ownership and run the project
    without relying on undocumented knowledge.
