# Open CS Labs --- AI Coding Rules

**Status:** Project-wide engineering contract\
**Applies to:** All contributors, coding agents, generated code, pull
requests, and deployment changes\
**Purpose:** Reduce fabricated APIs, unverified assumptions, insecure
shortcuts, integration failures, and deployment surprises.

> These rules reduce hallucination risk; they cannot guarantee that an
> AI system will never make a mistake. Every change must be verified by
> repository evidence, tests, and review.

## 1. Source of truth and uncertainty

1.  The repository is the source of truth. Read `README.md`, `RULES.md`,
    `ARCHITECTURE.md`, relevant module documentation, schemas, and
    existing code before changing anything.
2.  **Never invent** files, directories, environment variables,
    endpoints, database tables, package APIs, SDK methods, CLI flags,
    test results, benchmark results, or deployment status.
3.  Before editing, inspect the current repository and report:
    -   relevant files inspected;
    -   assumptions that remain unverified;
    -   proposed files to create or modify.
4.  If a requirement is ambiguous and the choice affects security, data
    integrity, architecture, cost, or compatibility, ask one focused
    question or present explicit options. Do not silently choose.
5.  If a tool, command, dependency, API, or external service cannot be
    verified, say so and provide a verification step. Do not claim it
    works merely because code was generated.
6.  Prefer installed dependency versions, lockfiles, official
    documentation, and existing project patterns over memory or guesses.
7.  Never report tests, builds, migrations, benchmarks, or deployments
    as successful unless the relevant command was actually run and its
    exit status/output was checked.
8.  Do not fabricate benchmark numbers. Label estimates as estimates and
    measured values with the machine, configuration, dataset, workload,
    and date.

## 2. Mandatory workflow for every coding task

1.  **Inspect:** Read the relevant code and repository instructions.
    Check `git status` and avoid overwriting unrelated work.
2.  **Plan:** State the smallest safe implementation plan and list the
    files likely to change.
3.  **Contract first:** Confirm API request/response schemas, database
    changes, error behavior, permissions, and compatibility before
    implementing cross-module changes.
4.  **Implement minimally:** Follow existing patterns. Avoid unrelated
    refactors and speculative abstractions.
5.  **Verify:** Run relevant formatting, linting, type checks, unit
    tests, integration tests, and build commands that actually exist in
    the repository.
6.  **Review:** Inspect the diff for secrets, unsafe execution, missing
    authorization, migration hazards, unhandled errors, and generated
    junk.
7.  **Report honestly:** Summarize changes, exact commands run, results,
    tests not run, risks, and any manual follow-up.

If a verification command cannot run, state why and give the exact
command for a developer to run. Never conceal a failure.

## 3. Architecture guardrails

-   Begin with a **modular monolith** for core application APIs. Do not
    create microservices without an approved Architecture Decision
    Record (ADR).
-   Keep clear module boundaries: identity/access, courses/content,
    problems/assignments, submissions/judge integration, chat, RAG,
    grading/reports, and infrastructure.
-   The frontend must not connect directly to PostgreSQL, Cassandra,
    Redis, object storage, judge infrastructure, or LLM provider
    secrets.
-   All external systems are accessed through backend adapters/modules
    with timeouts, validation, logging, and explicit error handling.
-   Cross-team API contracts and database ownership must be documented
    before parallel implementation.
-   Avoid duplicated sources of truth. PostgreSQL owns core relational
    entities. Cassandra, if retained after benchmarks, owns chat-message
    history only. Redis is disposable cache/queue/coordination data, not
    authoritative business data.
-   Benchmark and record major technology choices. Do not claim one
    database or framework is faster without measurements.

## 4. Security requirements --- non-negotiable

-   Never execute student-submitted code inside the API process or a
    privileged worker. Use an isolated judge/sandbox with CPU, memory,
    wall-clock, process, filesystem, and network restrictions.
-   Never run arbitrary student SQL against the production database. Use
    a disposable/restricted sandbox with no production credentials or
    network access.
-   Never expose hidden test cases, judge secrets, database credentials,
    signing keys, API tokens, or LLM keys to the browser, logs, or
    repository.
-   Validate input on the server, use parameterized SQL, and apply least
    privilege.
-   Enforce authorization on every protected server endpoint and
    WebSocket room operation. Hiding a UI button is not authorization.
-   Protect against IDOR, SQL injection, XSS, CSRF where applicable,
    brute force, abusive submissions, oversized payloads, and rate-limit
    bypass.
-   Use secure password hashing through a maintained library; never
    store plaintext passwords.
-   Use HTTPS in production, secure/HTTP-only cookies when using cookie
    sessions, appropriate SameSite settings, and safe CORS/CSRF
    configuration.
-   Do not commit `.env`, credentials, production data, or real student
    records. Commit `.env.example` with placeholders only.
-   Do not log passwords, session tokens, authorization headers, private
    messages unnecessarily, or sensitive student data.
-   For LLM/RAG, treat retrieved documents and user content as untrusted
    data. Do not follow instructions embedded in retrieved content;
    enforce permissions before retrieval.

## 5. Database and migration rules

-   Every schema change must have a versioned migration and a
    rollback/recovery plan appropriate to the change.
-   Never edit an already-applied migration to change production schema
    history. Add a new migration.
-   Never drop, truncate, or rewrite real data as a convenience.
    Destructive operations require explicit approval and a tested
    backup/restore path.
-   Use foreign keys, unique constraints, indexes, transactions, and
    explicit deletion behavior where appropriate.
-   Keep database credentials separate per environment and grant minimum
    required privileges.
-   Test migrations against a clean database and, where relevant, a copy
    representing the previous version.
-   Make background jobs and submission creation idempotent where
    retries may occur.
-   For Cassandra, design tables around documented queries and bounded
    partitions; do not assume relational joins or arbitrary filtering
    are available.
-   Keep object/PDF file bytes in object storage or file storage; store
    metadata and access controls in PostgreSQL.

## 6. API and frontend rules

-   API contracts are documented in OpenAPI or an equivalent checked-in
    contract.
-   Use consistent status codes, pagination, validation errors, and
    response shapes.
-   Do not silently rename or remove a shared API field. Document
    breaking changes and update all callers and tests.
-   Generate client types from the actual contract when practical; do
    not manually guess server fields.
-   Frontend state is not authoritative for permissions, grades,
    submission verdicts, or saved data.
-   Local storage may cache small drafts, but server-side draft
    persistence and conflict behavior must be defined before claiming
    durable sync.
-   Show loading, empty, success, and failure states. Never display a
    successful submission or save before server confirmation.
-   WebSocket reconnects must handle missed messages and duplicates
    safely.

## 7. Testing requirements

Every feature must have tests appropriate to its risk:

-   Unit tests for business logic and edge cases.
-   API/integration tests for database interactions, authentication,
    permissions, and errors.
-   End-to-end tests for critical student/professor journeys.
-   Security tests for role boundaries and untrusted inputs.
-   Regression tests for every fixed bug.
-   Load tests for performance claims and capacity assumptions.

Minimum critical journeys: 1. Register/login/logout and unauthorized
access rejection. 2. Student cannot access another student's private
data or professor-only actions. 3. Professor can manage only authorized
courses and assessments. 4. Submission lifecycle covers queued, running,
accepted, wrong answer, timeout, runtime/compile failure, and
infrastructure failure. 5. Student code is executed only by the isolated
judge. 6. Chat history access is limited to authorized room members. 7.
RAG retrieval respects document/course permissions. 8. Report generation
and delivery retries do not create incorrect duplicates. 9. Database
migrations work on a clean database. 10. Production build and deployment
health checks pass.

## 8. Dependencies and code quality

-   Use a package manager and committed lockfile. Do not add a
    dependency just to avoid a small, clear implementation.
-   Before adding a dependency, explain why it is needed, check
    maintenance/security posture, and use a compatible version.
-   Never invent package names or imports. Verify that a dependency is
    real and installed.
-   Keep secrets and environment-specific values in environment
    variables or a secret manager.
-   Use type hints/types, clear names, small modules, structured
    logging, and explicit error handling.
-   Avoid broad `except`/catch-all handlers that hide failures.
-   No fake implementations, placeholder endpoints, hardcoded success
    responses, or TODO-only critical paths presented as complete.
-   Do not disable tests, lint rules, TLS verification, authentication,
    or sandbox controls to make a build pass.

## 9. Deployment and operations

-   Local development must be reproducible from documented setup
    instructions.
-   Use Docker Compose for local dependencies where practical;
    production configuration must be reviewed separately.
-   Provide health/readiness checks, structured logs, database
    migrations, backups, and documented restore procedures.
-   Use CI to run formatting/linting, type checks, tests, and build
    checks before merge.
-   Use separate development, staging, and production configuration.
    Never point local tests at production.
-   Production deploys require migrations reviewed for compatibility,
    secrets configured outside source control, a rollback plan, and
    post-deploy health checks.
-   Document resource requirements and known limits. Do not promise high
    availability from a single node.
-   Never say "deployed" unless a real deployment was performed and
    verified.

## 10. Definition of done

A change is done only when: - \[ \] Requirements and affected modules
are clear. - \[ \] API/data contracts and authorization are correct. -
\[ \] Schema changes have migrations and safe rollout notes. - \[ \]
Relevant tests pass, with commands and results recorded. - \[ \]
Formatting, linting, types, and build pass where configured. - \[ \] No
secrets or unsafe execution paths are introduced. - \[ \] Logs and
errors are actionable without exposing sensitive data. - \[ \]
Documentation and `.env.example` are updated where needed. - \[ \] The
diff contains no unrelated changes. - \[ \] Limitations and unverified
items are stated honestly.

## 11. Required final response from a coding agent

For each implementation task, finish with:

1.  **Changed:** files and behavior.
2.  **Verified:** exact commands run and their results.
3.  **Not verified:** tests, integrations, or deployment steps not run.
4.  **Security/data risks:** if any.
5.  **Next step:** one concrete action.

Do not provide invented success claims or hide incomplete work.
