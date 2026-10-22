# ADR-0006: RAG storage and LLM provider

- **Status:** Proposed
- **Date:** 2026-10-22
- **Decision owners:** Chat/RAG/platform (Team 4)
- **Related areas:** `ARCHITECTURE.md` decision gates and flow E, `DATABASE_SCHEMA.md`
  section 2 (RAG), `API_CONTRACTS.md` section 7, `IMPLEMENTATION_PLAN.md` M5

## Context

Permission-aware doubt solving needs document ingestion, chunking, embeddings, retrieval,
and grounded answer generation. The provider must be chosen on privacy, cost, latency, and
quality, and all provider access must sit behind a backend adapter. No provider has been
evaluated.

## Decision drivers

- Security/privacy: retrieved documents and prompts must not leak; student data retentions
  terms matter.
- Cost and latency predictability.
- Simplicity: keep embeddings next to document metadata initially.

## Options considered

### Option A — pgvector for embeddings + a provider behind an adapter (leaning)
Benefits: keeps vector data with document metadata and permissions in PostgreSQL (ADR-0002),
one backup story, permission checks before retrieval. Drawbacks: provider still external.

### Option B — a dedicated vector database
Benefits: specialized vector features. Drawbacks: another datastore to operate and secure;
not justified before any measured retrieval workload.

## Evidence

**None.** No provider comparison or load test was run. This ADR stays `Proposed` until M5.

## Decision

Defer the provider and final vector-store choice to **M5** after a documented comparison.
Constraints that are binding now:

- Use pgvector in PostgreSQL initially; RAG chunk metadata and access policy stay in
  PostgreSQL.
- Provider access goes through a backend adapter with timeouts, quotas, and a safe fallback
  when the model is unavailable.
- Document permissions are enforced **before** retrieval; retrieved text is untrusted input.
- Provider keys are never exposed to the browser.

Not decided yet: provider, embedding model, chunking parameters, retention policy.

## Consequences

- Requires PostgreSQL with the pgvector extension (M1/M5 migration).
- Costs and latency must be measured and recorded in M5 before any capacity claim.
- `LLM_API_KEY` remains a commented placeholder until M5.

## Revisit conditions

- M5 provider comparison (quality/latency/privacy/cost).
- Retrieval performance measured on real course corpora.
