# Python project setup
How I stand up a new Python service from an empty folder. Commands and order only;
explanations live in the tool notes. Reference project: blasto.

## 1. Project - [uv](tooling/uv.md)
```bash
uv init <project-name>        # src layout: src/<project_name>/__init__.py
cd <project-name>
```
- Delete the `hello` `main()` in `__init__.py` and the `[project.scripts]` entry if unused.
- Add `.env` to `.gitignore` before any secrets exist.

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

## 6. GitHub + CI
Push (create an empty repo on GitHub first, no README/.gitignore/license):
```bash
git add .
git commit -m "Initial commit"
git branch -M main
git remote add origin git@github.com:<username>/<project-name>.git
git push -u origin main
```
CI workflow: TODO (S3)

## 7. Docker
TODO (S4)

## 8. Deploy
TODO (S5)
