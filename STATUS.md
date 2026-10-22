# Open CS Labs — Status

Status values are `NOT STARTED`, `IN PROGRESS`, `BLOCKED`, `DONE`.
`DONE` requires recorded evidence (commands actually run and their results), per `RULES.md`.
Milestone definitions live in `IMPLEMENTATION_PLAN.md`, which is the canonical numbering.

| Milestone | Description | Status | Evidence |
| --- | --- | --- | --- |
| M0 | Repository reconnaissance and decisions | DONE | See "M0 evidence" below |
| M1 | Foundation and first vertical slice | NOT STARTED | — |
| M2 | Problems, editor, and drafts | NOT STARTED | — |
| M3 | Course teaching workflows | NOT STARTED | — |
| M4 | Chat (incl. PostgreSQL vs Cassandra benchmark gate) | NOT STARTED | — |
| M5 | RAG doubt solving | NOT STARTED | — |
| M6 | Analytics, grading, and weekly reports | NOT STARTED | — |
| M7 | Hardening and deployment | NOT STARTED | — |

## Confirmed decisions

Recorded as ADRs in `docs/ADRs/`:

- Stack and toolchain: Python 3.12 + uv, Node 22 LTS + pnpm, React + TypeScript + Vite, FastAPI (ADR-0001).
- PostgreSQL is the authoritative store (ADR-0002).
- Cassandra is **conditional and not adopted**; benchmark gate at M4 (ADR-0003).
- Authentication: server-managed session cookie + CSRF (ADR-0004).
- Judge isolation, RAG provider, deployment topology: proposed, decided in later milestones.

## Open items / risks

- CI host assumed to be GitHub Actions; the repository has no remote yet, so the workflow is unverified until pushed.
- Exact patch versions of some tools were captured in ADR-0001; re-verify when the environment changes.

## M0 evidence

Repository was **not** under version control before M0 (`git status` returned
`fatal: not a git repository`). It was initialized with `git init -b main`.

Toolchain captured on 2026-10-22: Node v25.2.0 (default; `.nvmrc` pins 22.23.1),
pnpm 9.15.0, npm 11.6.2, Python 3.12.3, Git 2.43.0, Docker 29.9.1, Compose v2.40.3.
`uv` was installed during M0: version 0.13.5.

Commands run and observed results:

| Command | Result |
| --- | --- |
| `pnpm install` (+ `--frozen-lockfile`) | exit 0; `pnpm-lock.yaml` generated, lockfile in sync |
| `uv sync` (+ `--frozen`) | exit 0; `apps/api/uv.lock` generated, checked 13 packages |
| `uv run ruff check .` | `All checks passed!` |
| `uv run ruff format --check .` | `4 files already formatted` |
| `uv run mypy .` | `Success: no issues found in 3 source files` |
| `uv run pytest -q` | 1 passed |
| `pnpm exec prettier --check "**/*.{json,yml,yaml}"` | `All matched files use Prettier code style!` |
| `git check-ignore -v .env` | matched `.gitignore:25:.env` (env files ignored) |
| `git add -An apps/` | stages only the 6 intended files; `.venv/` and `.ruff_cache/` excluded |

Not verified: the GitHub Actions workflow has not run — the repository has no remote yet.
Do not report CI as passing until it is observed.
