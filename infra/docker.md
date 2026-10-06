# Docker
Docker packages and runs an application in a loosely isolated environment called a container. Containers are lightweight and contain everything needed to run the application, so they don't rely on what's installed on the host.

## Images
An image is a read-only template with instructions for creating a container. Often, an image is based on another image, with some additional customization.

We can build our own images or use ones built by others and published in a registry. To build an image, we write a `Dockerfile`, which defines the steps needed to create the image and run it. Each instruction in a Dockerfile creates a layer in the image.

Each layer contains a set of filesystem changes: additions, deletions, or modifications. When we change the Dockerfile and rebuild the image, only the layers that have changed are rebuilt. Layers let Docker reuse existing work, so the image isn't rebuilt from scratch every time. This is part of what makes images lightweight, small, and fast compared to other virtualization technologies.

## Compose
If several services need to run together, **Docker Compose** defines and runs multiple containers at once. For example, we create a Dockerfile for the backend, the frontend, and other services, and then use Docker Compose to run them all together.

## Commands
- `docker pull <image>` - Pull an image from a registry.
- `docker run <image>` - Run a container from an image. To run it in the background: `docker run -d <image>`. To map ports: `docker run -p <host_port>:<container_port> <image>`. To remove the container when it stops: `docker run --rm <image>`.
- `docker image ls` - List all images.
- `docker ps` - List all running containers. `docker ps -a` lists all containers, including stopped ones.
- `docker stop <container>` - Stop a running container.
- `docker container prune` - Remove all stopped containers.
- `docker image rm <image>` - Remove an image.
- `docker exec -it <container> <command>` - Run a command in a running container. For example, `docker exec -it <container> /bin/sh` opens a shell in the container.
- `docker build -t <image> <path>` - Build an image from the `Dockerfile` in the specified path.
- `docker compose up` - Run all services defined in the `docker-compose.yml` file.
- `docker compose down` - Stop and remove all services defined in the `docker-compose.yml` file.

## Best practices
### Non-root user
To limit what the process in the container is allowed to do, we run it as a non-root user. We create a new user with fewer rights and switch to it:
```Dockerfile
# Create an ordinary user named "app", so we do not run as "root", which has all rights
RUN useradd --create-home app

# Switch to the "app" user, so from now on everything runs as "app"
USER app
```

### Multi-stage build
A single-stage image can contain data the application doesn't need. For example, we need `uv` to install the dependencies, but not to run the application. So the good practice is to build the image in multiple stages: first we install the dependencies, then we copy only what is required into the final image. See the example below.

### Layer split
Docker builds an image layer by layer and caches the layers, so we split the layers where possible to avoid unnecessary rebuilds. For example, when some `.py` files change, `pyproject.toml` and `uv.lock` usually don't, so there is no need to reinstall the dependencies. Instead of `COPY . /app`, we first copy only `pyproject.toml` and `uv.lock`, then install the dependencies, and finally copy the rest of the files:
```Dockerfile
WORKDIR /app

COPY pyproject.toml uv.lock /app/

# Disable development dependencies
ENV UV_NO_DEV=1

# Install the dependencies, but not the project itself yet, since it changes frequently
RUN uv sync --locked --no-install-project

# Copy the project files
COPY . /app

# Install the project itself
RUN uv sync --locked
```

## Example
From `blasto`. First, we create a `Dockerfile` in the project root:
```Dockerfile
# Stage 1: install the dependencies
FROM python:3.14-slim AS installer

COPY --from=ghcr.io/astral-sh/uv:0.12.21 /uv /uvx /bin/

WORKDIR /app

COPY pyproject.toml uv.lock /app/

# Disable development dependencies
ENV UV_NO_DEV=1

# Install the dependencies, but not the project itself yet, since it changes frequently
RUN uv sync --locked --no-install-project

# Copy the project files
COPY . /app

# Install the project itself
RUN uv sync --locked


# Stage 2: run the application
FROM python:3.14-slim

WORKDIR /app

COPY --from=installer /app /app

# Put the virtual environment first in PATH, so commands like "uvicorn" and "python" are found there
ENV PATH="/app/.venv/bin:$PATH"

EXPOSE 8000

# Create an ordinary user named "app", so we do not run as "root", which has all rights
RUN useradd --create-home app

# Switch to the "app" user, so from now on everything runs as "app"
USER app

CMD ["uvicorn", "blasto.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

We also create a `.dockerignore`, so Docker doesn't copy unnecessary files into the image:
```
# Python
**/__pycache__/

# Virtual environment
.venv

# Tool caches
**/.pytest_cache/
**/.mypy_cache/
**/.ruff_cache/

# Secrets
.env

# Git
.git
.github

# Docker
Dockerfile
.dockerignore
```

We build the image with `docker build -t blasto:1.0 .` and run it with `docker run --rm -d -p 8000:8000 blasto:1.0`.
