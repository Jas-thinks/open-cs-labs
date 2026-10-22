# ADR-0007: Deployment topology and backup/restore

- **Status:** Proposed
- **Date:** 2026-10-22
- **Decision owners:** Integration/release owner (Team 4 platform)
- **Related areas:** `DEPLOYMENT.md`, `ARCHITECTURE.md` sections 8-9, `IMPLEMENTATION_PLAN.md` M7

## Context

Local development will use Docker Compose, but the staging and production topology,
secrets management, and backup/restore procedures are not yet decided or tested. No
environment has been provisioned.

## Decision drivers

- Operational burden for a student team and handover.
- Data safety for SQL sandbox and PostgreSQL.
- Cost and security of secrets management.

## Options considered

### Option A — Single small server + managed database
Benefits: low cost, simple mental model; managed DB gives backups. Drawbacks: limited
scale; must document the capacity ceiling honestly.

### Option B — Managed container platform + managed services
Benefits: easier scaling and deploys. Drawbacks: cost, more configuration surface.

## Evidence

**None.** No environment has been deployed. `DEPLOYMENT.md` section 10 requires a restore
test before any backup is considered verified; that has not happened.

## Decision

Defer the concrete topology to **M7**. Binding constraints now:

- Local development uses Docker Compose; `.env` is untracked, `.env.example` holds placeholders.
- Separate configuration/credentials for local, CI, staging, and production.
- Secrets live in the platform secret store in deployed environments, never in Git.
- PostgreSQL (and any adopted durable store) has backups and a **tested** restore before
  release readiness.
- Never claim a deployment succeeded without checking the target environment.

Not decided yet: host/platform, database hosting, DNS/TLS provider, resource sizes.

## Consequences

- CI cannot be validated until a remote host for the repository is chosen (see `STATUS.md`).
- M7 must record measured capacity limits and complete a restore drill.
- Judge and SQL sandbox networks are isolated from production data stores.

## Revisit conditions

- M7 environment selection.
- Measured load or cost data changes the single-server assumption.
