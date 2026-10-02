# ruff
An extremely fast Python linter (a tool that analyzes the code and looks for potential problems, bugs, or bad practices) and code formatter, written in Rust.

## Install
In a project, the best practice is to add it as a dev dependency: `uv add --dev ruff`. This pins the version in `uv.lock`, so my machine and CI run the same ruff.
It can also be installed globally as a tool: `uv tool install ruff` (handy outside projects, but the version isn't pinned per project).

## Run in cmd, and view the code in cmd during working
In a project, all commands are run with `uv run` in front (e.g. `uv run ruff check .`). Without it, they only work if ruff is installed as a global tool.

To check a specific file: `ruff check <path_to_file>`.
To check the whole directory: `ruff check <path_to_directory>`.
If we want **ruff** to check for issues in real time and show us the results in the terminal, we can use the `--watch` flag: `ruff check --watch <path_to_directory_or_file>`.

It can output the locations of the issues found, and the code with details of each issue. We can check the issue details in the **ruff documentation**, using the issue code.

To fix the fixable issues, run check with the `--fix` flag: `ruff check --fix <path_to_directory_or_file>`.
To see what fixes it is going to apply without applying them, add the `--diff` flag: `ruff check --fix --diff <path_to_directory_or_file>`.

To format the code: `ruff format <path_to_directory_or_file>`.
To only check the formatting without changing files (used in CI): `ruff format --check <path_to_directory_or_file>`.

The best practice is to run the linter first, fix the issues, and then run the formatter, because lint fixes can leave code that needs reformatting.

## Set up **ruff** for a project
We can configure **ruff** for the whole project in `pyproject.toml`. In particular, we can set which violations to look for, etc. To do this, we add a `[tool.ruff]` section to the `pyproject.toml` file. For example:

```toml
[tool.ruff]
lint.extend-select = ["PTH"]
```

The exact rules can be found in the **ruff documentation**.

We can also create a user-level config, e.g. `~/.config/ruff/ruff.toml`. It applies to all my projects, but only to those that have no ruff config of their own.
