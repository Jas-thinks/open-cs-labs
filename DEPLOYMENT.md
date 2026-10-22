# Open CS Labs — Development and Deployment Runbook

**Status:** Operational template. Fill in actual hostnames, secrets, resource sizes, and commands only after the repository and target environment are verified.

## 1. Environments

Maintain separate configuration and credentials for:
- local development;
- automated test/CI;
- staging;
- production.

Never point local tests at production. Use synthetic data in development and CI. Production data must not be copied into a developer laptop without an approved privacy-safe process.

## 2. Prerequisites

The project should document supported versions for:
- Git;
- Docker Engine and Docker Compose plugin;
- Node.js and package manager;
- Python runtime and package manager;
- optional database clients and load-test tools.

Do not guess versions. Pin them in project configuration and document the versions verified by CI.

## 3. Clean-clone setup acceptance

A developer should be able to:
1. clone the repository;
2. copy `.env.example` to a local `.env`;
3. fill only local development values;
4. start required dependencies using the documented Compose profile;
5. install frontend/backend dependencies from lockfiles;
6. run database migrations;
7. seed synthetic development data;
8. run backend and frontend;
9. run tests and build;
10. access health checks and complete a documented smoke-test journey.

The exact commands must be added after `docker-compose.yml`, package scripts, and backend configuration exist. Do not publish commands that have not been tested.

## 4. Configuration and secrets

- `.env.example` lists names, safe defaults, and descriptions, never real credentials.
- `.env` is ignored by Git.
- Production secrets come from the deployment platform's secret store.
- Validate required settings on startup and fail clearly if missing.
- Never log secret values.
- Keep separate least-privilege credentials for app runtime, migrations, judge, and reporting where appropriate.
- Rotate exposed credentials and remove them from active configuration; deleting a file from the latest commit is not sufficient if the secret entered Git history.

## 5. Local dependencies

Docker Compose may run PostgreSQL and other dependencies required for the selected milestone. Add Cassandra and Redis only when the current implementation needs them; use profiles where helpful to avoid requiring every service for a simple frontend task.

Do not mount production secrets or the Docker socket into untrusted code-execution containers. Judge infrastructure should be isolated from the application database and other sensitive services.

## 6. Database migration procedure

Before a migration:
1. review generated migration code manually;
2. identify locks, destructive operations, and backward compatibility;
3. test against a clean database and an upgrade copy;
4. confirm backup and restore readiness for persistent environments;
5. define rollout and recovery steps.

Production migrations require explicit human review. Prefer expand/contract migrations for changes incompatible with currently deployed code. Do not automatically drop tables or data during startup.

## 7. CI requirements

Pull-request CI should run:
- formatting/lint;
- type checks;
- unit tests;
- integration tests using disposable services;
- frontend production build;
- backend build/package validation;
- migration checks;
- security checks available in the project.

Protect the main branch with required checks and review. CI success is not equivalent to production deployment success.

## 8. Staging deployment checklist

- Use separate secrets and non-production data.
- Apply reviewed migrations.
- Verify readiness/liveness endpoints.
- Test login, role permissions, course access, submissions, chat, and private reports.
- Verify judge isolation and SQL sandbox restrictions.
- Confirm log redaction and metrics.
- Test object storage access policies.
- Test backup and restoration.
- Run a controlled load test.
- Record release commit, results, and known issues.

## 9. Production deployment checklist

- [ ] TLS/HTTPS configured.
- [ ] Secrets stored outside the repository.
- [ ] Database network access restricted.
- [ ] Runtime DB roles have least privilege.
- [ ] Database backup schedule and retention configured.
- [ ] Restore procedure recently tested.
- [ ] Migrations reviewed and compatible with rollout.
- [ ] Judge and SQL execution are isolated from production data.
- [ ] Authentication, authorization, rate limits, and upload limits enabled.
- [ ] Logs/metrics/alerts configured without sensitive payload leakage.
- [ ] Private PDFs and course documents require authorization.
- [ ] Health checks and post-deploy smoke tests pass.
- [ ] Rollback/recovery procedure is documented.
- [ ] Responsible maintainer and incident contact are identified.

## 10. Backups and recovery

Document for each persistent system:
- what is backed up;
- frequency and retention;
- encryption and access control;
- recovery point and recovery time objectives;
- who can restore;
- restore test date and result.

A successful backup job does not prove recoverability. Regularly restore into a safe isolated environment and verify data integrity. If Cassandra is adopted, include its replication, snapshot, restore, and schema migration procedures.

## 11. Monitoring

At minimum monitor:
- application health, HTTP error rate and p95 latency;
- WebSocket connection counts, delivery latency and errors;
- database connectivity, latency, disk and connection pools;
- submission queue depth, wait time and judge failures;
- background job retries and age;
- RAG provider latency/errors/cost where available;
- disk/object-storage capacity and backup outcomes.

Alerts should identify an actionable failure and its owner. Do not put secrets or private message content into metric labels.

## 12. Incident and rollback basics

When a deployment fails:
1. stop risky rollout or traffic shift;
2. capture relevant release ID, logs, metrics, and error context without secrets;
3. determine whether rollback is safe with the current schema;
4. roll back application code or apply a forward fix according to the migration plan;
5. verify health and critical journeys;
6. document root cause and add a regression test.

Never run a destructive rollback or restore over production data without explicit authorization and a verified recovery plan.
