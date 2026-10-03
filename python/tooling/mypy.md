# mypy
A static type checker for Python. It checks whether the code violates type annotations.

## Install
In a project, the best practice is to add it as a dev dependency: `uv add --dev mypy`. This pins the version in `uv.lock`, so my machine and CI run the same mypy.
It can also be installed globally as a tool: `uv tool install mypy` (handy outside projects, but the version isn't pinned per project).

## Run in cmd
In a project, all commands are run with `uv run` in front (e.g. `uv run mypy .`). Without it, they only work if mypy is installed as a global tool.

To check a specific file: `mypy <path_to_file>`.
To check the whole directory: `mypy <path_to_directory>`.

By default mypy checks only static functions (the ones that have type annotations). It does not check dynamic functions:

```python
# is not checked by mypy
def dynamic_add(a, b):
    return a + b

# is checked by mypy
def static_add(a: int, b: int) -> int:
    return a + b

result_static = static_add(1, "abc") # will be caught by mypy
result_dynamic = dynamic_add(1, "abc") # will not be caught by mypy, even though it causes a runtime error
```

However, with the `--strict` flag, every function without type annotations becomes an error, so I am forced to annotate everything — and then calls like dynamic_add(1, "abc") get caught too: `mypy --strict <path_to_directory_or_file>`

## Set up mypy for a project
We can configure mypy for the whole project in `pyproject.toml`. In particular, we can set the strict mode, etc. To do this, we add a `[tool.mypy]` section to the `pyproject.toml` file. For example:

```toml
[tool.mypy]
strict = true
```
Other useful options in `[tool.mypy]`:
- `files = ["src", "tests"]` - what to check, so plain `uv run mypy` works
- `plugins = ["pydantic.mypy"]` - makes mypy understand Pydantic models
- `[[tool.mypy.overrides]]` with `ignore_missing_imports = true` - silences a library that has no type hints

mypy can also be configured in `mypy.ini`; I use `pyproject.toml`.
