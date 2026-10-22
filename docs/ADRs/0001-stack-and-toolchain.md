# ADR-0001: Stack and toolchain baseline

- **Status:** Accepted
- **Date:** 2026-10-22
- **Decision owners:** M0 foundation (repository reconnaissance)
- **Related areas:** `ARCHITECTURE.md` section 2, `IMPLEMENTATION_PLAN.md` M0/M1, whole repository

## Context

The repository started as documentation only: no version control, no source code, no
lockfiles, no CI. M0 must pin a stack and toolchain before any implementation so four
teams build against the same versions. `ARCHITECTURE.md` section 2 proposed a stack but
marked it as unvalidated. Toolchain decisions were confirmed with the team and are
recorded here.

## Decision drivers

- Reproducible builds from lockfiles (`RULES.md` section 8).
- Familiar ecosystem for a 16-person student team with handover requirements.
- Minimal moving parts at the start (modular monolith, not microservices).
- Ability to run CI on every pull request.

## Environment verified on 2026-10-22 (evidence)

Captured by running the commands shown; these are measured values, not estimates.

| Tool | Version | Command |
| --- | --- | --- |
| Node.js (default active) | v25.2.0 | `node -v` |
| Node.js (installed LTS line, pinned) | v22.23.1 | `ls ~/.nvm/versions/node` |
| pnpm | 9.15.0 | `pnpm --version` |
| npm | 11.6.2 | `npm --version` |
| Python | 3.12.3 | `python3 -V` |
| Git | 2.43.0 | `git --version` |
| Docker | 29.9.1 (build dfc520d) | `docker --version` |
| Docker Compose | v2.40.3-desktop.1 | `docker compose version` |

Also present: `java`. Not installed: `uv` (added in M0), `psql` / local PostgreSQL,
`go`, `yarn`, `bun`. PostgreSQL will run via Docker Compose from M1.

## Options considered

### Option A — Proposed stack (chosen)
React + TypeScript + Vite; Tailwind CSS + shadcn/ui; React Router + TanStack Query;
Monaco editor; Python + FastAPI + Pydantic; SQLAlchemy 2 + Alembic; PostgreSQL
(+ pgvector); pytest/HTTPX/Vitest/React Testing Library/Playwright.

Benefits: matches the documented blueprint, broad ecosystem, good OpenAPI support on the
backend. Drawbacks: TypeScript/Pydantic duplication of types until a client is generated.

### Option B — Node/TypeScript end-to-end
Benefits: one language, shared types. Drawbacks: weaker fit for the RAG/Python tooling
expected in M5, and it diverges from the already-reviewed architecture, causing churn.

## Decision

Adopt the Option A stack and these toolchain pins:

- **Backend:** Python `>=3.12,<3.13` managed with **uv** (committed `uv.lock`), located at
  `apps/api` with a `src/` layout. Runtime dependencies (FastAPI, Pydantic, SQLAlchemy,
  Alembic, psycopg) are added and pinned in M1.
- **Frontend / monorepo:** **Node 22 LTS** (`.nvmrc` = `22.23.1`) with **pnpm**
  (`packageManager: pnpm@9.15.0`), workspace defined in `pnpm-workspace.yaml`.
- **Quality tools:** `ruff` (lint + format), `mypy` strict, `pytest` for the backend;
  `prettier` for JSON/YAML in M0 (ESLint/TypeScript added in M1 when code exists).
- **Local infrastructure:** Docker Compose (from M1).

Rationale for uv: single tool for interpreter management, dependency resolution, and a
committed lockfile, which `RULES.md` requires for reproducibility. It is the only new
tool added in M0; it must be installed before `uv sync` runs.

Not chosen: end-to-end TypeScript; Poetry (extra tool without the interpreter-management
benefit here); npm (pnpm is already installed and workspace-friendly).

## Consequences

- CI installs Node from `.nvmrc` and Python via `uv python install 3.12`.
- Local development uses Node 22; the machine's default Node 25 also satisfies
  `engines.node >=22.23.1`, but CI is authoritative.
- `uv` is not present on the current machine and must be installed
  (`curl -LsSf https://astral.sh/uv/install.sh | sh` or an equivalent verified method).
- Adding runtime dependencies requires updating `uv.lock` and citing the reason
  (`RULES.md` section 8).

## Revisit conditions

- A framework or runtime reaches a breaking/unsupported state.
- RAG or judge needs make the Python/TypeScript split costly.
- Team skill profile changes the maintenance calculus.
