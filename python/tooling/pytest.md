# pytest
A testing framework for writing tests for Python code.

## Install
In a project, the best practice is to add it as a dev dependency: `uv add --dev pytest`. This pins the version in `uv.lock`, so my machine and CI run the same pytest.
It can also be installed globally as a tool: `uv tool install pytest` (handy outside projects, but the version isn't pinned per project).

## Usage
For every file we want to test, we create a `test_<filename>.py` file. For each function we want to test, we create a `test_<function_name>` function in it. pytest finds these automatically by the `test_` prefix.

In a project, commands are run with `uv run` in front, from the project root:
- `uv run pytest` - run all tests in the project
- `uv run pytest tests/test_<filename>.py` - run one test file

### Unit tests
The smallest type of test, testing individual functions or methods. The goal is to ensure that we get the expected behavior from the tested component. By testing individual small components, we can catch bugs early and isolate them to specific snippets of code.

```python
import pytest


def test_add() -> None:
    assert add(2, 3) == 5


def test_divide() -> None:
    assert divide(10, 2) == 5
    with pytest.raises(ValueError, match="Cannot divide by zero"):
        divide(10, 0)
```

### Fixtures
Fixtures are functions that set up the test environment. They are executed before each test function that takes them as an argument, so we can use them to set up shared state or resources, like class instances, database connections, etc. For example, a fixture that creates an instance of a class:

```python
@pytest.fixture
def user_manager() -> UserManager:
    """Creates a fresh instance of UserManager before each test."""
    return UserManager()


def test_add_user(user_manager: UserManager) -> None:
    assert user_manager.add_user("john_doe", "john@example.com")
    assert user_manager.get_user("john_doe") == "john@example.com"
```

If we just create a global variable instead, it will be shared across tests and not reset in between: global objects are created once, so changes from one test leak into the next.

For cleanup after the test (teardown), the fixture uses `yield` instead of `return`; the code after `yield` runs when the test is done:

```python
@pytest.fixture
def db_connection() -> Iterator[Connection]:
    conn = connect()
    yield conn
    conn.close()
```

Fixtures shared by several test files go into `tests/conftest.py`; pytest loads it automatically, no import needed.

### Parameterized tests
In case we have many test cases, we can just create a matrix with these cases (the names in the decorator must match the function's parameter names):

```python
@pytest.mark.parametrize("number, expected", [
    (1, False),
    (2, True),
    (3, True),
    (10, False),
    (11, True),
])
def test_is_prime(number: int, expected: bool) -> None:
    assert is_prime(number) == expected
```

### Mocking
Frequently, there are parts of the code that rely on something that is not available in a testing environment: DB connections, external APIs, clients, etc. In these cases, we can use a mocking library to simulate the behavior of the external dependency.

Installing the mocking library: `uv add --dev pytest-mock`. A pytest plugin must be in the same environment as pytest, so for a pytest installed as a tool it's `uv tool install pytest --with pytest-mock`.

### FastAPI
To test FastAPI endpoints, we use `fastapi.testclient.TestClient`. It calls the app directly in memory, so no server needs to be running. It needs `httpx2` as a dev dependency (`uv add --dev httpx2`), otherwise Starlette shows a deprecation warning.

```python
import pytest
from fastapi.testclient import TestClient

from myapp.main import app


@pytest.fixture
def client() -> TestClient:
    return TestClient(app)


def test_read_main(client: TestClient) -> None:
    response = client.get("/")
    assert response.status_code == 200
    assert response.json() == {"message": "Hello World"}
```

The `client` fixture usually lives in `tests/conftest.py`, so every test file can use it.
