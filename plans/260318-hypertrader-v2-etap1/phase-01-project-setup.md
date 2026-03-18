---
phase: 1
title: Project Setup
status: pending
effort: 30m
---

# Phase 1: Project Setup

## Overview
Create project skeleton: directory structure, dependency files, all `__init__.py` files.

## Files to Create

```
HYPERTRADER/
├── src/
│   ├── __init__.py
│   ├── models/
│   │   └── __init__.py
│   ├── collectors/
│   │   └── __init__.py
│   ├── storage/
│   │   └── __init__.py
│   ├── events/
│   │   └── __init__.py
│   ├── server/
│   │   └── __init__.py
│   └── utils/
│       └── __init__.py
├── static/
├── config/
├── data/
├── pyproject.toml
└── requirements.txt
```

## Implementation Steps

### 1. `requirements.txt`
```
aiohttp>=3.9,<4
fastapi>=0.115,<1
uvicorn[standard]>=0.34,<1
websockets>=14,<15
pydantic>=2.10,<3
pydantic-settings>=2.7,<3
pyyaml>=6.0,<7
feedparser>=6.0,<7
```

### 2. `pyproject.toml`
```toml
[project]
name = "hypertrader"
version = "2.0.0"
requires-python = ">=3.12"
dependencies = [
    "aiohttp>=3.9,<4",
    "fastapi>=0.115,<1",
    "uvicorn[standard]>=0.34,<1",
    "websockets>=14,<15",
    "pydantic>=2.10,<3",
    "pydantic-settings>=2.7,<3",
    "pyyaml>=6.0,<7",
    "feedparser>=6.0,<7",
]

[project.scripts]
hypertrader = "src.main:main"
```

### 3. All `__init__.py` files -- empty

### 4. Create directories
- `static/`
- `config/`
- `data/` (add `.gitkeep`)

## TODO
- [ ] Create directory tree
- [ ] Write `pyproject.toml`
- [ ] Write `requirements.txt`
- [ ] Create all `__init__.py` files
- [ ] Add `data/.gitkeep`
- [ ] Run `pip install -e .` to verify

## Success Criteria
- `python -c "import src"` works
- All dependencies install without errors
