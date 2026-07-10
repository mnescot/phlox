# Phlox Backend

FastAPI backend for Phlox. See the top-level [README](../README.md) for setup and the
docs index, and [docs/ARCHITECTURE.md](../docs/ARCHITECTURE.md) for the system map.

```bash
uv sync
cp config.yml.example config.yml   # edit with your provider profiles
uv run uvicorn app.main:app --reload --port 8000
```

Lint and test (what CI runs):

```bash
uv sync --extra dev
uv run ruff check app tests
uv run pytest                      # add --cov=app for a coverage report
```
