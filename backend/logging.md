# logging
Logs are messages the application writes while it runs, so we can see what it is doing and trace bugs. Python has a built-in `logging` module for this.

## What to log
Whatever helps. Too many logs is noise, costs storage, and buries the line we need.

Some things must never be logged: passwords, tokens, keys, and personal data (names, emails, phone numbers).

## Where logs go
In containers, the convention is that the application writes its logs to the terminal output (standard output or standard error), not to files. Docker captures both, and we read them with `docker logs <container>` or `docker compose logs <service>`. The application doesn't manage log files itself.

So the problem with `print` is not where it writes. It is that `print` has no levels, no timestamps, and can't be turned down without editing the code. The `logging` module solves all three.

## Levels
Each log message has a level:

| Level | Meaning | Example |
|---|---|---|
| `DEBUG` | Details for tracing a bug | "Query returned 3 slots" |
| `INFO` | Normal events worth knowing | "GET /slots 200 in 12 ms" |
| `WARNING` | Something odd, but the app copes | "Retrying the database connection" |
| `ERROR` | Something failed | "Could not save the booking" |
| `CRITICAL` | The app can't continue | "Database unreachable at startup" |

The configured level (e.g. from a `LOG_LEVEL` setting) is a threshold: it shows that level and everything above it.
- `DEBUG` shows everything.
- `INFO` hides the debug messages.
- `WARNING` shows only problems.

Production usually runs at `INFO`; locally we use `DEBUG`.

## Example
Basic logging setup:
```python
import logging
import sys


def setup_logging(level: str) -> None:  # call once, at startup
    logging.basicConfig(  # configure the root logger, which all loggers report to
        level=level,  # the threshold, e.g. "INFO": lower levels are hidden
        format="%(asctime)s %(levelname)s %(name)s %(message)s",  # time, level, logger name, message
        stream=sys.stdout,  # write to stdout (the default would be stderr)
    )
```

Then, in `main`, we configure logging before the app starts, and create `main`'s own logger:
```python
setup_logging(settings.log_level)  # configure logging before anything logs
logger = logging.getLogger(
    __name__
)  # a logger named "blasto.main"; the name shows in every line
```

Every other module creates its own logger the same way:
```python
logger = logging.getLogger(__name__)    # __name__ is the module's name, e.g. "blasto.routes.slots"
```

We can also write a middleware that logs every request, for example:
```python
import logging  # to create this file's logger
import time  # to measure how long a request takes
from collections.abc import Awaitable, Callable  # types for the middleware's call_next

from fastapi import (
    FastAPI,
    Request,
    Response,
)

from blasto.config import settings
from blasto.logging_setup import setup_logging

setup_logging(settings.log_level)  # configure logging before anything logs
logger = logging.getLogger(
    __name__
)  # a logger named "blasto.main"; the name shows in every line


app = FastAPI()


@app.middleware("http")  # run this function around every HTTP request
async def log_requests(
    request: Request,
    call_next: Callable[
        [Request], Awaitable[Response]
    ],  # passes the request on to the endpoint
) -> Response:
    start = time.perf_counter()  # a precise clock, made for measuring durations
    response = await call_next(request)  # let the endpoint handle the request
    duration_ms = (time.perf_counter() - start) * 1000  # elapsed time in milliseconds
    logger.info(  # an INFO line: hidden when LOG_LEVEL is WARNING or higher
        "%s %s %s %.1fms",  # placeholders, filled in by logging only if the line is shown
        request.method,  # e.g. GET
        request.url.path,  # e.g. /health
        response.status_code,  # e.g. 200
        duration_ms,  # e.g. 0.8
    )
    return response  # send the response back to the client
```
