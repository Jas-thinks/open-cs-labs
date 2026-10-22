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

- CI is GitHub Actions and green on `main` (run `39743425740`). Two non-blocking warnings
  remain: the third-party actions target Node 20 (deprecated on runners) and `ubuntu-latest`
  will migrate to Ubuntu 26 in October 2026. Bumping action versions is a small follow-up.
- Exact patch versions of some tools were captured in ADR-0001; re-verify when the
  environment changes.
- `uv` is installed outside the repository (`~/.local/bin`). CI pins 0.13.5; a developer must
  install it separately.

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

### M0 closeout

Remote: `https://github.com/Jas-thinks/open-cs-labs` (public). `main` tracks `origin/main`.

| Check | Command | Result |
| --- | --- | --- |
| CI on `4e5af5b` | `gh run view 39743425740` | **success** — `Backend (lint, types, tests)` and `Workspace (format check)` both green; all steps completed |
| pnpm under Node 22 | `corepack pnpm --version` | `9.15.0` (exit 0) — documents the README fix |
| `.commandcode/` ignored | `git check-ignore -v .commandcode/taste/taste.md` | matched `.gitignore:30:.commandcode/` |
| Backend re-verified | `uv lock --check` / `ruff check` / `ruff format --check` / `mypy` / `pytest -q` | lock current; all checks pass |
| Workspace re-verified | `pnpm exec prettier --check "**/*.{json,yml,yaml}"` | conforms |

CI run: https://github.com/Jas-thinks/open-cs-labs/actions/runs/39743425740

Not verified / deferred: Docker Compose and any application behavior (there is still no
application code). Judge, RAG provider, and deployment topology remain open decisions
(ADR-0005/0006/0007).
