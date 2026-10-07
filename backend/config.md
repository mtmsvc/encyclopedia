## Config

We can read the config manually, but an easier way is `pydantic-settings`. It replaces all the manual reading and gives us a structure in which every variable is checked by type and by allowed values (`Literal`); other checks, like a length limit, have to be declared explicitly with `Field`. The structure is created at startup, so if a check fails, the app stops at startup. It reads the values both from the system environment and from the `.env` file.

```python
from typing import Literal

from pydantic_settings import (
    BaseSettings,
    SettingsConfigDict,
)


class Settings(
    BaseSettings
):  # each field is filled from the env var of the same name; it is case-insensitive
    model_config = SettingsConfigDict(
        env_file=".env"
    )  # also read values from the .env file, if it exists; system environment variables override file values

    env: Literal["local", "production"] = (
        "local"  # ENV: where the app runs; default "local"
    )
    log_level: Literal["DEBUG", "INFO", "WARNING", "ERROR"] = (
        "INFO"  # LOG_LEVEL: how much to log
    )
    database_url: str = "postgresql+asyncpg://blasto:blasto@localhost:5432/blasto"  # DATABASE_URL: used in S7; local default


settings = Settings()  # read everything once; other files import this object
```
