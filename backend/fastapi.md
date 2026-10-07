# FastAPI
A modern, fast (high-performance) web framework for building APIs with Python.

## Under the hood
- The **event loop** - runs on one thread and switches between requests: while one request waits (e.g. for the database), it handles another. Many async requests share that one thread, so if we block it, everything freezes.
- **uvicorn** is an **ASGI** server. ASGI is the standard interface between async Python servers and frameworks. It runs the event loop and hands each request to your app.
- **FastAPI** is built on **Starlette** (routing, requests, responses) plus **Pydantic** (validation).

When going deeper, watch:
- https://www.youtube.com/watch?v=rvFsGRvj9jo
- https://www.youtube.com/watch?v=nYAMtzAbNN8
- https://www.youtube.com/watch?v=kmJz8w5ij8Y

## Install and run
To install FastAPI, run `uv add "fastapi[standard]"`. Plain `fastapi` is only the framework: it has no `fastapi` CLI and no server, so it can't run by itself. The `[standard]` extra adds both the CLI and `uvicorn`, the server that actually runs the app.

### With the `fastapi` CLI
- `uv run fastapi dev {path_to_app}` - development mode: restarts automatically on every file save, listens only on `127.0.0.1` (only my machine can reach it).
- `uv run fastapi run {path_to_app}` - production mode: no automatic restart, listens on `0.0.0.0` (reachable from other machines, needed in Docker).

### With `uvicorn` directly
This is what I use. On the server (in the Dockerfile):
```bash
uvicorn blasto.main:app --host 0.0.0.0 --port 8000 --no-access-log
```
- `blasto.main:app` - the module (`blasto/main.py`), then the variable in it that holds the FastAPI app.
- `--host 0.0.0.0` - accept connections from outside the container.
- `--port 8000` - the port to listen on.
- `--no-access-log` - turn off uvicorn's own log line per request, since our logging middleware already logs each request, with its duration.

Locally:
```bash
uv run uvicorn blasto.main:app --reload
```
- `--reload` - restart the server automatically whenever a code file changes, so I don't have to stop and start it after every edit. Changes to `.env` are not watched: after editing `.env`, restart by hand.

### From a main block
Another local option: start the server inside the app file itself, in a main block (`if __name__ == "__main__":`):
```python
import uvicorn

if __name__ == "__main__":
    uvicorn.run(app, host="0.0.0.0", port=8000)
```
Then the file runs like any script: `uv run python {path_to_app}`. Running by path works with absolute imports (`from blasto.config import settings`); with relative imports (`from .config import settings`) it breaks, and I need `uv run -m blasto.main` instead.

## Config
We can read the config manually, but an easier way is `pydantic-settings`. It replaces all the manual reading and gives us a structure in which every variable is checked by type and by allowed values (`Literal`); other checks, like a length limit, have to be declared explicitly with `Field`. The structure is created at startup, so if a check fails, the app stops at startup. It reads the values both from the system environment and from the `.env` file.

```python
from typing import Literal

from pydantic_settings import BaseSettings, SettingsConfigDict


class Settings(BaseSettings):  # each field is filled from the env var of the same name; case-insensitive
    model_config = SettingsConfigDict(env_file=".env")  # also read .env, if it exists; real env vars win

    env: Literal["local", "production"] = "local"  # ENV: where the app runs
    log_level: Literal["DEBUG", "INFO", "WARNING", "ERROR"] = "INFO"  # LOG_LEVEL: how much to log
    database_url: str = "postgresql+asyncpg://blasto:blasto@localhost:5432/blasto"  # DATABASE_URL: local default


settings = Settings()  # read everything once; other files import this object
```

## Middleware
A middleware is a function that runs around every request: code before the endpoint, then the endpoint itself (`call_next`), then code after it. Used for things every request needs, like logging or timing.

```python
@app.middleware("http")
async def my_middleware(
    request: Request, call_next: Callable[[Request], Awaitable[Response]]
) -> Response:
    # before the endpoint
    response = await call_next(request)
    # after the endpoint
    return response
```
A request-logging middleware is in `python/logging.md`.
