# Progress Log

## 2026-03-18 — Session 1

### Tasks completed
- Phase 1: Project Setup

### Files created
- `pyproject.toml` — build config, dependencies, entry point
- `requirements.txt` — pip dependencies
- `.gitignore` — Python, venv, data exclusions
- `src/__init__.py` + 6 subpackage `__init__.py` (models, collectors, storage, events, server, utils)
- `src/main.py` — stub entry point
- `data/.gitkeep`, `static/.gitkeep`, `config/.gitkeep`

### Decisions made
- Removed `websockets` direct dependency — `uvicorn[standard]` pulls it transitively
- Added stub `src/main.py` so `[project.scripts]` entry point doesn't break before Phase 9
- Python 3.12 venv confirmed working, `pip install -e .` passes

### Issues
- Initial venv created with Python 3.9 (system default) — fixed by using `/opt/homebrew/bin/python3.12`
- `setuptools.backends._legacy` build backend doesn't exist — fixed to `setuptools.build_meta`

### Next session
- Phase 2 (Models + Config), Phase 3 (Utils), Phase 4 (EventBus) — all unblocked, can be done in parallel
