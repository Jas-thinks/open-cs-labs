# Open CS Labs — Testing and Benchmark Strategy

## 1. Principles

- Correctness and authorization come before performance.
- Tests must be deterministic where practical and must not use production accounts/data.
- No benchmark number is accepted without an actual run, configuration, raw output, and machine details.
- Load tests run only against a controlled test environment with synthetic data.
- Every fixed bug gets a regression test when feasible.
- Test scripts must fail with a non-zero exit code on required failure.

## 2. Test layers

### Unit tests
Test pure business rules, validation, grading calculations, status transitions, cursor handling, and error normalization without external services.

### API/integration tests
Run against disposable PostgreSQL and the required test dependencies. Verify persistence, transactions, authorization, migration compatibility, and dependency failure behavior.

### Frontend tests
Use component tests for forms/loading/error states and Playwright for critical user journeys.

### Security tests
Test unauthorized access, cross-course access, role escalation, IDOR, input size limits, rate limits, SQL injection attempts, XSS-safe rendering, CSRF behavior when applicable, hidden-test leakage, report access, and RAG permissions.

### Load/benchmark tests
Use synthetic datasets and a documented workload. Separate application latency from database latency and end-to-end user-perceived latency.

## 3. Critical acceptance scenarios

### Identity and access
- Valid registration/login/logout.
- Invalid credentials fail safely.
- A student cannot grant themselves professor/admin roles.
- Anonymous users cannot access protected data.
- A student cannot read another student's private draft, submissions, or reports.
- Professor actions are limited to courses they are authorized to manage.

### Course content
- Only authorized staff can create/edit announcements and assignments.
- Private resources are inaccessible to users without permission.
- Upload size/type limits are enforced.

### Submissions and judging
- Valid submission persists before the API reports it as queued.
- Duplicate request with the same idempotency key does not create duplicate effects.
- All expected verdicts are tested: accepted, wrong answer, time limit, memory limit where supported, runtime error, compile error, judge/infrastructure failure.
- Hidden tests are never returned to student-facing endpoints.
- Timeouts and sandbox limits are enforced by the judge environment, not merely by frontend code.
- Queue retry and worker restart do not silently lose or incorrectly duplicate submissions.

### SQL sandbox
- Student SQL cannot access production credentials or production network.
- Restricted role cannot read/write outside the permitted sandbox.
- Long-running queries time out.
- Results are bounded by row/byte limits.
- Sandbox reset/cleanup is verified.

### Chat
- Only room members can join and send.
- Oversized or abusive message rates are rejected.
- Persistence failure does not produce a false successful-send acknowledgement.
- Retry/deduplication works.
- Reconnect and history pagination recover missed messages.
- If multiple API instances are used, cross-instance delivery is tested.

### RAG
- Retrieval respects course and document permissions.
- Prompt injection inside retrieved text does not override system policy.
- Provider timeout/rate limit returns a safe, useful error.
- Answers provide sources when the retrieval contract supports them.
- Private documents do not leak across courses.

### Grading and reports
- Relative-grade calculation handles empty data, small cohorts, zero variance, ties, missing/excused work, and policy changes.
- Recommendations are not silently finalized.
- Overrides require an authorized actor and reason.
- Report generation retry does not create inconsistent duplicates.
- Private report downloads require authorization.
- Delivery failure is recorded and retryable.

## 4. Suggested test commands

These are proposed conventions, not claims that scripts already exist. Confirm actual project scripts before running.

Backend examples:
```bash
pytest
pytest -q
```

Frontend examples, after `package.json` scripts are defined:
```bash
npm run test
npm run build
```

Repository CI should expose stable, documented commands. Do not invent commands in automation; inspect `pyproject.toml`, `package.json`, and CI configuration first.

## 5. Benchmark protocol

Record for each run:
- commit hash and date;
- CPU, memory, OS, runtime/container versions;
- database topology and configuration;
- dataset generation seed, size, room/user distribution, payload sizes;
- warm-up, ramp-up, concurrency, duration, repeats;
- throughput, p50/p95/p99 latency, error rate, CPU, memory, disk and network;
- raw output and exact command used.

Run at least one warm-up and multiple measured repetitions. Report variance and test limitations. Compare systems on equal resource limits and equivalent workloads.

## 6. Chat storage comparison: PostgreSQL vs Cassandra

Use identical synthetic message datasets and workload profiles. Measure separately:
1. database write acknowledgement latency;
2. room history query latency;
3. end-to-end sender-to-recipient WebSocket delivery latency;
4. throughput and errors at increasing concurrency;
5. recovery after process/database restart;
6. operational effort for setup, backups, monitoring, upgrades, and restore.

Do not compare one database on a powerful multi-node cluster with another on a tiny laptop and call the result a fair comparison. If the intended deployment is a single small server, measure that reality too.

Decision criteria must be agreed before the test. Consider performance, consistency needs, failure behavior, team expertise, hosting cost, operational burden, and expected scale. Cassandra remains conditional until the evidence and ownership plan are reviewed.

## 7. Release gates

A release candidate is not ready until:
- critical unit and integration tests pass;
- frontend/backend builds pass;
- migrations work on a clean test database;
- authorization/security tests pass;
- core end-to-end student and professor flows pass;
- judge and SQL isolation checks pass;
- backups and restore are tested for persistent data stores;
- staging smoke tests pass;
- known failures and untested areas are documented.

Do not mark a gate passed if its command was not run.
