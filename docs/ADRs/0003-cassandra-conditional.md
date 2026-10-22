# ADR-0003: Cassandra for chat history (conditional)

- **Status:** Proposed
- **Date:** 2026-10-22
- **Decision owners:** Chat/RAG/platform (Team 4)
- **Related areas:** `ARCHITECTURE.md` decision gates and sections 4-5, `DATABASE_SCHEMA.md`
  section 5, `TESTING.md` section 6, ADR-0002

## Context

`ARCHITECTURE.md` lists Cassandra as a *conditional* store for chat-message history only.
It must not be adopted on assumption. No benchmark has been run, so justification does not
currently exist. The team also has no operational experience with Cassandra recorded.

## Decision drivers

- Data integrity: room identity, membership, and permissions must stay authoritative in
  PostgreSQL (ADR-0002).
- Operational burden and backup/restore for a student team.
- Realistic expected scale, which is not yet measured.

## Options considered

### Option A — PostgreSQL for chat history (default until proven otherwise)
Benefits: one datastore, existing backups, transactional membership checks, familiar.
Drawbacks: write-heavy append workloads may need tuning; unverified.

### Option B — Cassandra for chat history
Benefits: potentially high append throughput with bounded-partition reads. Drawbacks:
extra operational surface (topology, repairs, snapshots, migrations), no joins, eventual
consistency concerns, and no benchmark evidence on our hardware or workload.

## Evidence

**None.** No benchmark has been run. `ARCHITECTURE.md` section 10 and `TESTING.md` section 6
define the required protocol (same dataset/workload, database latency separated from
end-to-end WebSocket delivery, warm-up + repeats, operator cost included). Until that is
executed and recorded, this ADR stays `Proposed` and Cassandra is not adopted.

## Decision

Cassandra is **not adopted** at this time.

- PostgreSQL owns chat history initially.
- The Cassandra decision is revisited at **M4**, after the benchmark gate, via a new or
  updated ADR with reproducible results.
- If ever adopted, Cassandra owns message history only; PostgreSQL remains authoritative
  for room identity, membership, courses, and permissions.

## Consequences

- No Cassandra container or driver until M4.
- The chat persistence layer (M4) must be implemented behind an adapter so a store can be
  swapped only if the benchmark justifies it.
- Postgres chat tables must still be designed around documented access patterns (bounded
  room history reads), not arbitrary filtering.

## Revisit conditions

- A reproducible benchmark showing a meaningful, operationally sustainable advantage.
- A measured scale requirement that PostgreSQL cannot meet on the target deployment.
