# ADR-0005: Code execution isolation strategy

- **Status:** Proposed
- **Date:** 2026-10-22
- **Decision owners:** Judge/practice (Team 3)
- **Related areas:** `ARCHITECTURE.md` decision gates and flows B/C, `RULES.md` section 4,
  `IMPLEMENTATION_PLAN.md` M2

## Context

DSA submissions and practice SQL must never execute inside the API process or against
production data. `ARCHITECTURE.md` lists Judge0 "or another reviewed isolated judge" and
marks the choice conditional. The specific judge product, resource limits, and network
isolation are not yet validated.

## Decision drivers

- Security: CPU, memory, wall-clock, process, filesystem, and network restrictions.
- Operational feasibility for a student team.
- Distinguishing infrastructure failure from a wrong answer.

## Options considered

### Option A — Judge0 (self-hosted, isolated)
Benefits: existing multi-language support and a documented API. Drawbacks: deployment,
resource limits, and security posture must be validated before accepting student code.

### Option B — A custom minimal sandbox runner
Benefits: full control over limits. Drawbacks: high risk of subtle escape/limit bugs for a
small team; more work than the milestone allows.

## Evidence

**None.** No judge has been deployed or tested. This ADR stays `Proposed` until M2
validates deployment and isolation.

## Decision

Defer the concrete judge choice to **M2**. Binding constraints now:

- Untrusted code executes only in an isolated judge/sandbox, never in the API process.
- The judge is reached through an authenticated, rate-limited backend adapter.
- Student SQL runs only against a disposable/restricted sandbox with no production
  credentials or network access, with statement timeouts and bounded results.

Not decided yet: Judge0 vs alternative; sandbox runtime; exact limit values.

## Consequences

- M2 must produce an ADR update with the chosen judge, tested limits, and isolation evidence.
- The submission state machine and verdict normalization are backend-owned regardless of judge.
- Judge infrastructure is kept off the application database network.

## Revisit conditions

- M2 validation results (deployment, escape/limit testing).
- A judge's maintenance or security posture degrades.
