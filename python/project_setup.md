# Python project setup
How I stand up a new Python service from an empty folder. Commands and order only;
explanations live in the tool notes. Reference project: blasto.

## 1. Project - [uv](tooling/uv.md)
```bash
uv init <project-name>        # src layout: src/<project_name>/__init__.py
cd <project-name>
```
- Delete the `hello` `main()` in `__init__.py` and the `[project.scripts]` entry if unused.
- Add `.gitignore`. Files like:
```
__pycache__/
*.py[oc]
build/
dist/
wheels/
*.egg-info
.venv
.pytest_cache/
.mypy_cache/
.ruff_cache/
.env
```

## 2. First endpoint - [FastAPI](../backend/fastapi.md)
```bash
uv add "fastapi[standard]"
```
`src/<project_name>/main.py`:
```python
from fastapi import FastAPI

app = FastAPI()

@app.get("/health")
def health() -> dict[str, str]:
    return {"status": "ok"}
```
```bash
uv run fastapi dev src/<project_name>/main.py   # check /health and /docs
```

## 3. Lint and format - [ruff](tooling/ruff.md)
`pyproject.toml`:
```toml
[tool.ruff]
lint.extend-select = ["I"]   # import sorting, off by default
```
```bash
uv add --dev ruff
uv run ruff check --fix .
uv run ruff format .
```

## 4. Type checking - [mypy](tooling/mypy.md)
```bash
uv add --dev mypy
```
`pyproject.toml`:
```toml
[tool.mypy]
strict = true
```
Optionally, add `files = ["src", "tests"]` under `strict = true`; then plain
`uv run mypy` checks only those directories. Without `files`, pass the path:
```bash
uv run mypy .
```

## 5. Tests - [pytest](tooling/pytest.md)
```bash
uv add --dev pytest httpx2      # httpx2: needed by FastAPI's TestClient
```
```
tests/
├── conftest.py      # shared fixtures, loaded automatically
├── test_main.py     # API tests (TestClient)
└── test_utils.py    # unit tests
```
`tests/conftest.py`:
```python
import pytest
from fastapi.testclient import TestClient

from <project_name>.main import app


@pytest.fixture
def client() -> TestClient:
    return TestClient(app)
```
All checks, from the project root, before every commit:
```bash
uv run ruff check --fix .
uv run ruff format .
uv run mypy src tests
uv run pytest
```

I create a file to always run all checks before pushing:
`scripts/check.sh` (then `chmod +x scripts/check.sh`):
```bash
#!/usr/bin/env bash
set -e
uv run ruff check --fix .
uv run ruff format .
uv run mypy src tests
uv run pytest
```
## 6. GitHub + CI - [git](../infra/git.md), [GitHub Actions](../infra/github-actions.md)
Push (create an empty repo on GitHub first, no README/.gitignore/license):
```bash
git add .
git commit -m "Initial commit"
git branch -M main
git remote add origin git@github.com:<username>/<project-name>.git
git push -u origin main
```

`.github/workflows/checks.yml`:
```yaml
name: checks

on:
  pull_request:
    branches: [main]
  push:
    branches: [main]

jobs:
  checks:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@3d3c42e5aac5ba805825da76410c181273ba90b1 # v7.0.1

      - name: Set up Python
        uses: actions/setup-python@5fda3b95a4ea91299a34e894583c3862153e4b97 # v7.0.0
        with:
          python-version-file: ".python-version"

      - name: Install uv
        uses: astral-sh/setup-uv@c771a70e6277c0a99b617c7a806ffedaca235ff9 # v9.0.0
        with:
          enable-cache: true
          version: "<local uv --version>"

      - name: Install the project
        run: uv sync --locked

      - name: Lint
        run: uv run ruff check .

      - name: Format
        run: uv run ruff format --check .

      - name: Typecheck
        run: uv run mypy src tests

      - name: Test
        run: uv run pytest
```

Protect `main`: Settings → Branches → rule for `main` →
"Require a pull request before merging" + "Require status checks to pass" → `checks`.

Workflow from then on: branch → PR → green `checks` → merge.
Local checks before pushing: `./scripts/check.sh` (the four commands from section 5).

## 7. Docker
TODO (S4)

## 8. Deploy
TODO (S5)
