# Open CS Labs

Open CS Labs is a course learning and practice platform for SQL and DSA, with automated evaluation, course content, student/professor dashboards, room chat, RAG-based doubt support, and reporting.

## Start here

1. Read `CLAUDE.md` if using Claude Code.
2. Read `RULES.md` and `ARCHITECTURE.md`.
3. Review `IMPLEMENTATION_PLAN.md` and start with M0.
4. Use `API_CONTRACTS.md` and `DATABASE_SCHEMA.md` before changing shared interfaces or data models.
5. Follow `TESTING.md` and `DEPLOYMENT.md`.

## Current status

This repository documentation is a blueprint, not proof that the application has been implemented or deployed. Verify actual code, dependencies, commands, and test results before making status claims.

**M0 (repository foundation) is complete:** version control, toolchain pinning, quality-tool
configuration, environment template, ADRs, and a CI skeleton exist. **There is no
application code yet** — the FastAPI app, health endpoints, database, migrations, and
frontend arrive in M1. See `STATUS.md` and `docs/ADRs/`.

## Local development setup (verified)

Prerequisites: Git, Node.js 22, Python 3.12, and [uv](https://docs.astral.sh/uv/).

Node 22 ships `corepack` but **not** `pnpm`. Enable the pinned version once (`packageManager`
in `package.json` is `pnpm@9.15.0`), or call it through corepack without enabling shims:

```bash
nvm use             # reads .nvmrc -> Node 22.23.1
corepack enable     # one-time; provides `pnpm` at the pinned version
# or, without enabling shims:
corepack pnpm --version   # verified: prints 9.15.0
```

Install uv if it is missing:

```bash
curl -LsSf https://astral.sh/uv/install.sh | sh
```

Then, from the repository root:

```bash
cp .env.example .env              # fill in local values; .env is git-ignored
pnpm install --frozen-lockfile    # JS workspace dependencies
cd apps/api && uv sync --frozen   # Python dependencies for the backend
```

Quality checks (backend, from `apps/api`; workspace, from the repository root):

```bash
uv run ruff check . && uv run ruff format --check . && uv run mypy . && uv run pytest -q
pnpm run format:check
```

These commands were executed during M0 and produced exit code 0. CI runs the same checks
(`.github/workflows/ci.yml`); the workflow itself is unverified until this repository is
pushed to a remote.

> Workspace note: `pnpm run lint|typecheck|test|build` recurse over workspace packages. There
> are none until `apps/web` lands in M1, so those scripts are currently no-ops.

> Troubleshooting: if `pnpm install` reports that devDependencies were skipped, the shell
> has `NODE_ENV=production` set. Run `NODE_ENV=development pnpm install --frozen-lockfile`.

## Proposed product areas

- Student identity and course access.
- Announcements, readings, lectures, assignments.
- Monaco-based SQL/DSA practice.
- Isolated DSA judging and SQL sandboxing.
- Drafts and submission history.
- Course chat with authorized history.
- Permission-aware RAG doubt solving.
- Progress analytics, professor grading tools, and weekly reports.

## Security baseline

Never execute untrusted code inside the main API process. Never run student SQL against production. Keep secrets out of Git. Enforce authorization server-side on every protected endpoint and room action.

## Documentation

- `CLAUDE.md` — Claude Code entry instructions.
- `RULES.md` — engineering and safety rules.
- `ARCHITECTURE.md` — system boundaries and deployment blueprint.
- `API_CONTRACTS.md` — proposed HTTP/WebSocket contracts.
- `DATABASE_SCHEMA.md` — logical data ownership and schema proposal.
- `TESTING.md` — test layers, acceptance scenarios, benchmark protocol.
- `DEPLOYMENT.md` — environment and release runbook.
- `IMPLEMENTATION_PLAN.md` — milestones and team ownership.
- `docs/ADRs/ADR_TEMPLATE.md` — decision record template.
