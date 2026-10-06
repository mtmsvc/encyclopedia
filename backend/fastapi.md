# FastAPI
A modern, fast (high-performance) web framework for building APIs with Python.

## Under the hood
- The **event loop** - the app is basically just one or several threads, that accept some task (requests). Several async requests can share one thread, and if we block it, everything freezes.
- **uvicorn** is an **ASGI** server. ASGI is the standard interface between async Python servers and frameworks. It runs the event loop and hands each request to your app.
- **FastAPI** is built on **Starlette** (routing, requests, responses) plus **Pydantic** (validation).

When going deeper, watch:
- https://www.youtube.com/watch?v=rvFsGRvj9jo
- https://www.youtube.com/watch?v=nYAMtzAbNN8
- https://www.youtube.com/watch?v=kmJz8w5ij8Y
- 
## Install and run
To install FastAPI, run `uv add "fastapi[standard]"`. Plain `fastapi` is only the framework: it has no `fastapi` CLI and no server, so it can't run by itself. The `[standard]` extra adds both the CLI and `uvicorn`, the server that actually runs the app.

To run the app:
- `uv run fastapi dev {path_to_app}` - development mode: auto-reloads on every file save, listens only on `127.0.0.1` (only my machine can reach it).
- `uv run fastapi run {path_to_app}` - production mode: no reload, listens on `0.0.0.0` (reachable from other machines, needed in Docker).

On servers I run the server with `uvicorn` command, but on local machines, I prefer to put the server start in a main block (`if __name__ == "__main__":`), where I start the server with `uvicorn`:

```python
import uvicorn

if __name__ == "__main__":
    uvicorn.run(app, host="0.0.0.0", port=8000)
```

This way I can run the main file as usual with `uv run python {path_to_app}`. Running by path works with absolute imports (`from blasto.config import settings`); with relative imports (`from .config import settings`) it breaks, and I need `uv run -m blasto.main` instead.
