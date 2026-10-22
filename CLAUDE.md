# Claude Code Instructions — Open CS Labs

Read this file at the start of every task. It is the repository entry point for Claude Code.

## Required reading order
1. `RULES.md` — non-negotiable engineering, security, and verification rules.
2. `ARCHITECTURE.md` — system boundaries and deployment blueprint.
3. `IMPLEMENTATION_PLAN.md` — current milestone and task acceptance criteria.
4. The relevant sections of `API_CONTRACTS.md`, `DATABASE_SCHEMA.md`, `TESTING.md`, and `DEPLOYMENT.md`.
5. Existing code, migrations, lockfiles, tests, and Git status.

If any referenced file does not exist yet, say so and proceed only with what can be verified. Do not invent its contents.

## Core operating rules
- The repository and checked-in contracts are the source of truth. Existing implementation and migrations may reveal drift; report drift instead of silently choosing one.
- Never invent package APIs, files, environment variables, endpoints, schema fields, CLI options, test results, benchmark numbers, or deployment status.
- Before making changes, inspect `git status`, relevant files, current dependency versions, and project scripts. Preserve unrelated user changes.
- For non-trivial work, provide a concise plan, list likely files to change, then implement the smallest coherent change.
- If a decision affects security, data integrity, a public API, architecture, cost, or compatibility and the contracts do not settle it, ask a focused question or propose options before coding.
- Do not implement the whole project in one pass. Work in one milestone/task with explicit acceptance criteria.
- Do not create speculative abstractions or unrelated refactors.
- Do not add a dependency without explaining why and verifying its compatibility and maintenance status.
- Do not claim that a command passed unless you ran it and checked the exit code and output.
- If unable to run a test, state exactly why and provide the exact command needed to verify it.
- Do not fabricate benchmark or load-test results. Record configuration and measured results separately from estimates.

## Required task workflow
1. Inspect repository and current branch/status.
2. Identify relevant contract sections and acceptance criteria.
3. Summarize plan and risks.
4. Implement.
5. Add or update tests, migrations, API docs, and `.env.example` as applicable.
6. Run available formatting, lint, type-check, tests, and build commands.
7. Review the diff for secrets, unsafe execution, missing authorization, migrations, and unrelated changes.
8. Report:
   - changed files and behavior;
   - exact commands run and outcomes;
   - checks not run and why;
   - known risks or contract drift;
   - next concrete step.

## Project guardrails
- Start as a modular monolith. Do not introduce microservices without an approved ADR.
- PostgreSQL is the authoritative store for identity, course membership, assignments, submissions, grading, document metadata, and report metadata.
- Cassandra is a **conditional** store for chat-message history only. Do not assume it is required for the first working release; implement only after the benchmark/operations decision in `ARCHITECTURE.md` is approved.
- Redis is optional and must have a named purpose before it is added.
- Never execute student code in the API process. Use an isolated judge/sandbox.
- Never run student SQL against production data or with production credentials.
- Do not expose hidden tests, private reports, secrets, or cross-course RAG documents.
- Every protected REST route and WebSocket room action requires server-side authorization.
- Persist submission state before returning a submission ID; retries must not create accidental duplicate submissions.
- RAG documents and retrieved text are untrusted input, not instructions.
- Relative grades are recommendations; final grades require authorized professor review/approval and audit history.
- Do not use undocumented/private LeetCode endpoints or scraping as a core dependency.

## Contracts and drift
- `API_CONTRACTS.md` describes intended API behavior. Update it when a public contract changes.
- `DATABASE_SCHEMA.md` describes logical ownership and intended entities. Actual migrations are executable schema truth; keep both synchronized.
- `TESTING.md` defines minimum verification. Add regression tests for fixed bugs.
- `DEPLOYMENT.md` is the deployment runbook. Never claim deployment is complete without checking the target environment.
- Important decisions go in `docs/ADRs/` using the ADR template.

## First task for a fresh repository
Do not immediately build features. Inspect the repository and report:
1. what already exists;
2. what is missing;
3. toolchain/runtime versions found;
4. current tests/build status (only if actually run);
5. proposed smallest M0 setup task.
Wait for approval before making large structural changes.
