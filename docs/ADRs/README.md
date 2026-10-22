# Architecture Decision Records

Significant decisions are recorded here using [`ADR_TEMPLATE.md`](./ADR_TEMPLATE.md).
An ADR is `Proposed` until reviewed; conditional technology choices remain `Proposed`
and must not be treated as adopted (see `ARCHITECTURE.md` decision gates).

| ADR | Title | Status |
| --- | --- | --- |
| [0001](./0001-stack-and-toolchain.md) | Stack and toolchain baseline | Accepted |
| [0002](./0002-postgresql-authoritative.md) | PostgreSQL as the authoritative datastore | Accepted |
| [0003](./0003-cassandra-conditional.md) | Cassandra for chat history (conditional) | Proposed |
| [0004](./0004-authentication-session-cookies.md) | Authentication via server-managed session cookies | Accepted |
| [0005](./0005-judge-isolation.md) | Code execution isolation strategy | Proposed |
| [0006](./0006-rag-storage-and-provider.md) | RAG storage and LLM provider | Proposed |
| [0007](./0007-deployment-topology.md) | Deployment topology and backup/restore | Proposed |

## Notes

- ADR-0003, ADR-0005, ADR-0006 and ADR-0007 are deliberately `Proposed`: they depend on
  benchmark or environment evidence that does not exist yet. Do not describe them as final.
- M0 scope is repository foundation only; no application code exists as of ADR-0001.
