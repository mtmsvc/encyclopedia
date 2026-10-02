# uv
Python package manager, a replacement for pip. It is a lot faster than pip and is built with Rust.
It basically controls the whole project: the dependencies are declared in `pyproject.toml`, and their exact versions are pinned in `uv.lock`, from which uv can easily recreate the environment, even if we simply delete the actual folder of packages (`.venv`) from our project.

## Related files in the project
- `.venv` - virtual environment folder, storing the actual packages. Not committed to git.
- `.python-version` - the Python version the project uses.
- `pyproject.toml` - configuration file for the project (name, version, Python version, dependencies, dev dependencies, runnable commands, etc.)
- `uv.lock` - lock file with the exact versions of all dependencies, including dependencies of dependencies. Committed to git.

## Main commands I used
- `uv init <project-name>` - create a new project
- `uv add <package>` - install a package and record it in `pyproject.toml` and `uv.lock` (a runtime dependency, required to run the project)
- `uv add --dev <package>` - install a package as a dev dependency (recorded in `pyproject.toml` under `[dependency-groups]` and in `uv.lock`, but used only for development, like the linter/formatter `ruff`, the type checker `mypy` and the test runner `pytest`; skipped with `uv sync --no-dev`, e.g. in the production Docker image)
- `uv tool install <package>` - install a package as a tool (available everywhere for my user, in its own isolated environment, not tied to any project and not recorded in `uv.lock`)
- `uv remove <package>` - remove a package
- `uv tree` - show the dependency tree
- `uv run <command>` - run any command inside the project's environment, e.g. `uv run python file.py` or `uv run pytest`; syncs `.venv` first, no activation needed
- `uv run <script-name>` - run a command defined in `pyproject.toml` under `[project.scripts]`, which calls a specific function (`"package.file:function"`)
- `uv run -m <package>.<file>` - run a file inside the package as a module (same as `uv run python -m ...`), so relative imports inside the package work
- `uv sync` - create or update `.venv` so it matches `uv.lock` exactly
- `uv cache clean` - remove all cached packages from the local cache (uv keeps the installed packages in the cache to avoid reinstalling them)
