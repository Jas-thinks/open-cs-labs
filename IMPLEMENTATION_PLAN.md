# Open CS Labs — Implementation Plan

This plan deliberately breaks the project into reviewable milestones. Reorder based on course deadlines and team skill, but keep dependencies and acceptance criteria explicit.

## Working agreements

- One shared repository and a protected main branch.
- Every task has an owner, acceptance criteria, tests, and a small PR.
- API and database contracts are reviewed before teams implement against them.
- Integrate continuously; do not let four teams build isolated systems until the end.
- A milestone is complete only when acceptance checks were run and recorded.
- Keep a `STATUS.md` or project board current; do not claim unfinished work is done.

## M0 — Repository reconnaissance and decisions

**Goal:** establish the real starting point before generating code.

Tasks:
- Inspect current repository, Git status, available tools, existing code and lockfiles.
- Confirm team experience and runtime versions.
- Review `RULES.md`, `ARCHITECTURE.md`, API and schema proposals.
- Decide frontend/backend versions and package managers.
- Record initial stack decisions as ADRs in `docs/ADRs/` (template already exists at
  `docs/ADRs/ADR_TEMPLATE.md`; index at `docs/ADRs/README.md`).
- Agree code style, branch/PR rules, API error format, IDs, timestamps, and auth model.
- Define local setup and CI commands based on actual scripts.

Acceptance:
- A clean setup plan is documented.
- No existing work is overwritten.
- Decisions marked conditional remain explicitly conditional.
- Team owners and review process are agreed.

## M1 — Foundation and first vertical slice

**Goal:** prove frontend, backend, database, tests, and local setup work together.

Tasks:
- Initialize React + TypeScript frontend.
- Initialize FastAPI backend with health endpoints and OpenAPI.
- Configure PostgreSQL, SQLAlchemy, Alembic, and test database.
- Add environment validation and `.env.example`.
- Add formatter/linter/type/test/build commands and CI.
- Implement user registration/login/session/logout and authorization tests.
- Implement course listing and a basic student dashboard using real APIs.
- Add synthetic seed data.

Acceptance:
- New developer can follow the verified clean-clone instructions.
- Migration applies to a clean database.
- Register/login/course-list journey works end to end.
- Unauthenticated and unauthorized access is rejected.
- CI runs required checks and reports actual outcomes.

## M2 — Problems, editor, and drafts

**Goal:** deliver a real practice experience without executing untrusted code in the API.

Tasks:
- Problem catalog and problem detail APIs.
- Monaco editor integration and language selection.
- Draft cache and server-save behavior with explicit conflict semantics.
- Problem examples and hidden-test storage permissions.
- Submission schema, state machine, idempotency, and status endpoint.
- Integrate a reviewed isolated DSA judge.
- Implement disposable/restricted SQL practice sandbox.

Acceptance:
- Student can open a problem, edit code, save a draft, submit, and see a real persisted state.
- Judge handles representative success and failure verdicts.
- Hidden tests remain private.
- SQL sandbox cannot access production data.
- Timeout, payload, and rate limits are tested.

## M3 — Course teaching workflows

**Goal:** professors can manage course activity safely.

Tasks:
- Course membership and professor permissions.
- Announcements and learning resources.
- Assignment creation, due dates, problem links, and scores.
- Student submission/assignment views.
- Audit trail for sensitive grading actions.

Acceptance:
- Authorized professor can manage their course.
- Students cannot perform professor actions or access another course's private content.
- Critical flows have API and end-to-end tests.

## M4 — Chat

**Goal:** reliable authorized room chat with recoverable history.

Tasks:
- Define room/membership model and WebSocket event contract.
- Implement authentication, room authorization, message limits, IDs, persistence policy, and pagination.
- Benchmark PostgreSQL and Cassandra against equivalent synthetic workloads.
- Record an ADR deciding whether Cassandra is justified for target deployment.
- If approved, implement Cassandra history adapter and operations plan.
- Implement reconnect/backfill and cross-instance fan-out if deployment uses multiple API instances.

Acceptance:
- Only room members can join/read/send.
- Failed persistence does not appear as a successful send.
- Duplicate retry behavior and reconnect history are tested.
- Benchmark results are reproducible and include resource/configuration details.
- Cassandra is not adopted solely on unverified assumptions.

## M5 — RAG doubt solving

**Goal:** provide grounded course-aware answers.

Tasks:
- Select an LLM/embedding provider based on privacy, cost, latency, and quality.
- Ingest approved PDFs/text and preserve source/course/visibility metadata.
- Chunk documents and store embeddings in pgvector.
- Implement permission-aware retrieval, answer citations, provider timeout, quotas, and feedback.
- Test prompt injection and cross-course access.

Acceptance:
- Unauthorized documents are never retrieved for a user.
- Answers can cite source material when relevant sources exist.
- Provider failures degrade safely.
- Costs and latency are measured in a documented test.

## M6 — Analytics, grading, and weekly reports

**Goal:** support professor review without silently finalizing grades.

Tasks:
- Define event/metric semantics and weekly reporting windows.
- Student dashboard aggregates and course-level analytics.
- Configurable relative-grade policies and versioned grading runs.
- Edge-case handling: empty/small cohort, zero variance, ties, missing/excused work.
- Professor approval and audited manual overrides.
- Background PDF generation, private storage, delivery status and retry.

Acceptance:
- Metrics reconcile to authoritative source data.
- Recommendations are explainable and not automatically final.
- Report access is authorized and private.
- Retries are idempotent and failures are visible.

## M7 — Hardening and deployment

**Goal:** make the product operable and handover-ready.

Tasks:
- Staging environment and production configuration.
- Security review, dependency review, rate limits, upload restrictions, logging redaction.
- Load tests for API, chat, database, RAG and judge queue.
- Database backups and restore drill.
- Health checks, metrics, alerts, CI release gates, rollback/runbook.
- Complete README, API docs, architecture decisions, troubleshooting and student-team handover.

Acceptance:
- Staging smoke tests pass.
- Restore is demonstrated, not merely configured.
- Known capacity limits and benchmark conditions are documented.
- Deployment is verified in the target environment.
- A new maintainer can run the project using documentation alone.

## Team ownership proposal (16 people)

| Team | Primary ownership | Integration dependency |
|---|---|---|
| Team 1 — Frontend | React UI, navigation, Monaco, dashboards, chat UI | API contracts and UX states |
| Team 2 — Core backend | Auth, courses, content, PostgreSQL, API/OpenAPI | All modules |
| Team 3 — Judge/practice | DSA judge adapter, SQL sandbox, submission lifecycle | Backend contracts and security |
| Team 4 — Chat/RAG/platform | Chat, Cassandra evaluation, RAG, report jobs, observability | Core data ownership and frontend contracts |

Assign one integration/release owner and one reviewer for schema/API changes. Share the same test commands and CI. Team ownership does not mean teams can independently change shared contracts.

## Per-task template

For each task, record:
- Context and user-visible behavior.
- In-scope and out-of-scope work.
- Contract/schema changes.
- Security/authorization considerations.
- Acceptance criteria.
- Tests and exact commands.
- Dependencies and owner.
- Rollback/migration notes where relevant.

## Status tracking

Update this file or a linked issue tracker after each milestone. Record actual state as `NOT STARTED`, `IN PROGRESS`, `BLOCKED`, or `DONE`; `DONE` requires evidence from tests/review.
