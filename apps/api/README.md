# Open CS Labs — Backend (`apps/api`)

FastAPI modular monolith. **M0 stage: tooling configuration only** — there is no
runnable application yet. The FastAPI app, `/health` endpoints, SQLAlchemy models,
and Alembic migrations arrive in M1.

## Toolchain

- Python 3.12 (pinned in the repository-root `.python-version`)
- [uv](https://docs.astral.sh/uv/) for dependency management and the committed `uv.lock`
- `ruff` (lint + format), `mypy` (strict types), `pytest`

## Commands

Run from this directory:

```bash
uv sync --frozen            # install exactly what uv.lock pins
uv run ruff check .         # lint
uv run ruff format --check .  # formatting check (use `ruff format .` to apply)
uv run mypy .               # strict type check
uv run pytest -q            # tests
```

The source layout is `src/open_cs_labs_api/`. Application code is added in M1.
