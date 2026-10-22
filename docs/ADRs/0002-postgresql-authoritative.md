# ADR-0002: PostgreSQL as the authoritative datastore

- **Status:** Accepted
- **Date:** 2026-10-22
- **Decision owners:** Core backend (Team 2)
- **Related areas:** `DATABASE_SCHEMA.md`, `ARCHITECTURE.md` sections 4-5, ADR-0003

## Context

The platform needs transactional, constraint-enforced storage for identity, course
membership, assignments, submissions, grading, and document/report metadata. Multiple
teams depend on these entities, so ownership must be unambiguous.

## Decision drivers

- Correctness and data integrity (foreign keys, unique constraints, transactions).
- Security: hidden test data and private reports must be protected.
- Team familiarity and simple local setup.
- Avoid duplicated sources of truth.

## Options considered

### Option A — PostgreSQL as the single authoritative relational store (chosen)
Benefits: strong constraints and transactions, one place to reason about consistency,
pgvector extends it for RAG. Drawbacks: chat-scale writes are not its strength (addressed
in ADR-0003).

### Option B — Polyglot with multiple authoritative stores
Benefits: possible per-workload tuning. Drawbacks: consistency, ownership, and operational
burden across a small student team; not justified without evidence.

## Evidence

No benchmark was run. This decision is architectural, not performance-based. Performance
claims about chat workloads are explicitly deferred to the ADR-0003 benchmark gate.

## Decision

PostgreSQL is the authoritative store for: users, sessions, roles, courses, memberships,
announcements, learning resources, file metadata, problems, examples, assignments,
submissions and results, drafts, scores, grading policies/runs/recommendations/final
grades, audit events, RAG document/chunk metadata (with pgvector embeddings), and report
metadata/jobs/deliveries.

Not chosen: multiple authoritative relational stores; storing business records only in
Redis or an object store.

## Consequences

- Every schema change is an Alembic migration (M1+), tested on a clean database.
- Hidden test content is stored and access-controlled separately from public examples.
- Object storage holds bytes; PostgreSQL holds metadata and access policy.
- Backup/restore procedures apply here first (see ADR-0007).

## Revisit conditions

- Measured throughput/consistency limits under real load (M4, M7).
- A concrete query pattern that cannot be served acceptably, backed by a benchmark.
