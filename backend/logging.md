# logging
Logs are messages the application writes while it runs, so we can see what it is doing and trace bugs. Python has a built-in `logging` module for this.

## What to log
Whatever helps. Too many logs is noise, costs storage, and buries the line we need.

Some things must never be logged: passwords, tokens, keys, and personal data (names, emails, phone numbers).

## Where logs go
In containers, the convention is that the application writes its logs to standard output (stdout), the same place `print` writes to. Docker captures that output, and we read it with `docker logs <container>` or `docker compose logs <service>`. The application doesn't manage log files itself.

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
