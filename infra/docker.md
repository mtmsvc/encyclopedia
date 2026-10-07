# Docker
Docker packages and runs an application in a loosely isolated environment called a container. Containers are lightweight and contain everything needed to run the application, so they don't rely on what's installed on the host.

## Images
An image is a read-only template with instructions for creating a container. Often, an image is based on another image, with some additional customization.

We can build our own images or use ones built by others and published in a registry. To build an image, we write a `Dockerfile`, which defines the steps needed to create the image and run it. Each instruction in a Dockerfile creates a layer in the image.

Each layer contains a set of filesystem changes: additions, deletions, or modifications. When we change the Dockerfile and rebuild the image, only the layers that have changed are rebuilt. Layers let Docker reuse existing work, so the image isn't rebuilt from scratch every time. This is part of what makes images lightweight, small, and fast compared to other virtualization technologies.

## Volumes
A container's files are temporary: when the container is removed or recreated, everything written inside it is lost. To keep data, we store it outside the container, in a volume.

- **Named volume** - storage managed by Docker. We only give it a name; Docker decides where it is stored on the host. Used for data the application creates, like database files or certificates.
- **Bind mount** - a file or folder from the host, shown inside the container at a path we choose. The container reads the host's file directly, so when we change the file on the host, the container sees the change. Used for files we write ourselves, like configuration files.

```bash
docker run -v mydata:/data <image>                         # named volume "mydata", at /data in the container
docker run -v "$(pwd)/config.txt:/app/config.txt" <image>  # bind mount of a file from the current folder; pwd means "print working directory"
```

## Compose
If several containers need to run together, **Docker Compose** defines and runs them from one file, usually `compose.yml`. For example, for a backend and a database, we describe each one as a service, and Compose starts them all with one command.

```yaml
services:
  app:
    image: <app-image>
    ports:
      - "8000:8000"
    restart: unless-stopped

  db:
    image: <database-image>
    volumes:
      - db_data:/data
    restart: unless-stopped

volumes:
  db_data:
```
- **`services`** - one entry per container.
- **`image`** - which image the container runs.
- **`ports`** - `host:container`; makes the container reachable from outside the host.
- **`volumes`** (in a service) - attaches a named volume or a bind mount.
- **`volumes`** (at the end) - declares the named volumes.
- **`restart: unless-stopped`** - starts the container again after a crash or a reboot, unless we stopped it ourselves.

Compose connects all the containers in the file to their own private network. On that network, each container can reach the others by their service name: here, `app` can connect to the database at `db`. A container that only other containers talk to doesn't need `ports` at all.

Values in the file can come from a `.env` file in the same folder: `image: <app-image>:${TAG}` takes `TAG` from a line like `TAG=1.0` in `.env`.

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
- `docker login <registry>` - Log in to a registry, e.g. `docker login ghcr.io`.
- `docker push <image>` - Upload an image to a registry. The image name must start with the registry, e.g. `ghcr.io/<owner>/<image>:<tag>`.
- `docker compose up -d` - Create or update all services in `compose.yml` and start them in the background.
- `docker compose pull` - Download the newest versions of the services' images.
- `docker compose ps` - List the services' containers.
- `docker compose logs -f <service>` - Follow a service's logs.
- `docker compose restart <service>` - Restart a service, e.g. after changing its config file.
- `docker compose down` - Stop and remove all services in `compose.yml`. Named volumes are kept.

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

On the server, `blasto` runs behind Caddy, a web server that handles HTTPS and forwards requests to the app:
```yaml
services:
  app:
    image: ghcr.io/<owner>/blasto:${IMAGE_TAG}
    restart: unless-stopped

  caddy:
    image: caddy:2
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./Caddyfile:/etc/caddy/Caddyfile
      - caddy_data:/data

volumes:
  caddy_data:
```
Only Caddy has `ports`; it reaches the app at `app:8000` over the Compose network. The Caddyfile is a bind mount, so we edit it on the server. The certificates are kept in the named volume `caddy_data`, so they survive restarts.
