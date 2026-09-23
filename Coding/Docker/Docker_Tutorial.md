# Docker Tutorial

A practical, beginner-to-intermediate guide to Docker. Work through it top to bottom, or jump to the section you need.

## Table of Contents

1. [What Is Docker?](#what-is-docker)
2. [Core Concepts](#core-concepts)
3. [Installing Docker](#installing-docker)
4. [Your First Container](#your-first-container)
5. [Working with Images](#working-with-images)
6. [Managing Containers](#managing-containers)
7. [Writing a Dockerfile](#writing-a-dockerfile)
8. [Data Persistence: Volumes and Bind Mounts](#data-persistence-volumes-and-bind-mounts)
9. [Networking](#networking)
10. [Docker Compose](#docker-compose)
11. [Best Practices](#best-practices)
12. [Common Commands Cheat Sheet](#common-commands-cheat-sheet)
13. [Troubleshooting](#troubleshooting)

---

## What Is Docker?

Docker is a platform for building, shipping, and running applications inside **containers**. A container packages your app together with its dependencies, so it runs the same way on your laptop, a teammate's machine, or a production server.

Unlike a virtual machine, a container shares the host operating system's kernel instead of running a full guest OS. That makes containers lightweight, fast to start, and efficient with resources.

**Why it matters:**

- "It works on my machine" problems mostly go away.
- Environments are reproducible and version-controlled.
- Apps start in seconds, not minutes.
- You can run many isolated services on one host.

---

## Core Concepts

| Term | What it means |
|------|---------------|
| **Image** | A read-only template with your app and its dependencies. Think of it as a snapshot or blueprint. |
| **Container** | A running (or stopped) instance of an image. |
| **Dockerfile** | A text file with instructions to build an image. |
| **Registry** | A store for images. [Docker Hub](https://hub.docker.com) is the default public registry. |
| **Volume** | Docker-managed storage that persists data outside a container's lifecycle. |
| **Docker Compose** | A tool to define and run multi-container apps using a single YAML file. |

The mental model: a **Dockerfile** builds an **image**, and an **image** runs as a **container**.

---

## Installing Docker

The easiest way to get started on Windows and macOS is **Docker Desktop**.

- **Windows / macOS:** Download Docker Desktop from the [official install page](https://docs.docker.com/get-docker/) and run the installer.
- **Linux:** Install Docker Engine following the [Linux install docs](https://docs.docker.com/engine/install/) for your distribution.

Verify the install:

```bash
docker --version
docker run hello-world
```

If `hello-world` prints a welcome message, Docker is working.

---

## Your First Container

Run an Nginx web server in one command:

```bash
docker run -d -p 8080:80 --name my-web nginx
```

Breaking that down:

- `run` — create and start a container.
- `-d` — detached mode (runs in the background).
- `-p 8080:80` — map host port 8080 to container port 80.
- `--name my-web` — give the container a friendly name.
- `nginx` — the image to use (pulled from Docker Hub if not local).

Open <http://localhost:8080> in your browser and you'll see the Nginx welcome page.

Stop and remove it:

```bash
docker stop my-web
docker rm my-web
```

---

## Working with Images

```bash
# Search Docker Hub
docker search python

# Pull an image
docker pull python:3.12-slim

# List local images
docker images

# Remove an image
docker rmi python:3.12-slim
```

Image names follow the pattern `repository:tag`. If you omit the tag, Docker defaults to `latest`. Pinning a specific tag (like `python:3.12-slim`) makes builds reproducible.

---

## Managing Containers

```bash
# List running containers
docker ps

# List all containers (including stopped)
docker ps -a

# View logs
docker logs my-web

# Follow logs live
docker logs -f my-web

# Open a shell inside a running container
docker exec -it my-web bash

# Stop / start / restart
docker stop my-web
docker start my-web
docker restart my-web

# Remove a container
docker rm my-web

# Remove all stopped containers
docker container prune
```

The `-it` flags combine interactive mode (`-i`) with a pseudo-terminal (`-t`), which you need for shells.

---

## Writing a Dockerfile

A `Dockerfile` defines how to build your image. Here's a simple example for a Python app:

```dockerfile
# Start from an official base image
FROM python:3.12-slim

# Set the working directory inside the container
WORKDIR /app

# Copy dependency list first (better layer caching)
COPY requirements.txt .

# Install dependencies
RUN pip install --no-cache-dir -r requirements.txt

# Copy the rest of the app
COPY . .

# Document the port the app listens on
EXPOSE 8000

# Command to run when the container starts
CMD ["python", "app.py"]
```

Build and run it:

```bash
docker build -t my-python-app .
docker run -d -p 8000:8000 --name app my-python-app
```

The `-t` flag tags the image with a name. The `.` tells Docker to use the current directory as the build context.

**Tip:** Copying `requirements.txt` before the rest of the code lets Docker cache the dependency layer. Dependencies only reinstall when that file changes, which speeds up rebuilds.

---

## Data Persistence: Volumes and Bind Mounts

Containers are ephemeral. When you remove one, its internal data is gone. Use volumes or bind mounts to keep data around.

**Named volume (Docker-managed):**

```bash
docker volume create mydata
docker run -d -v mydata:/var/lib/data --name app my-python-app
```

**Bind mount (maps a host folder into the container):**

```bash
docker run -d -v C:\CodingDojo\project:/app --name dev my-python-app
```

Bind mounts are great for development because code changes on your host show up instantly in the container. Named volumes are better for production data like databases.

```bash
# List volumes
docker volume ls

# Remove unused volumes
docker volume prune
```

---

## Networking

By default, containers on the same user-defined network can reach each other by name.

```bash
# Create a network
docker network create app-net

# Run containers on it
docker run -d --network app-net --name db postgres
docker run -d --network app-net --name web my-python-app
```

Now `web` can connect to the database using the hostname `db`. This is the foundation for multi-service apps.

```bash
# List networks
docker network ls

# Inspect a network
docker network inspect app-net
```

---

## Docker Compose

For multi-container apps, Compose lets you define everything in one file. Create `docker-compose.yml`:

```yaml
services:
  web:
    build: .
    ports:
      - "8000:8000"
    depends_on:
      - db
    environment:
      DATABASE_URL: postgres://user:pass@db:5432/mydb

  db:
    image: postgres:16
    environment:
      POSTGRES_USER: user
      POSTGRES_PASSWORD: pass
      POSTGRES_DB: mydb
    volumes:
      - db-data:/var/lib/postgresql/data

volumes:
  db-data:
```

Manage the whole stack with:

```bash
# Build and start everything (detached)
docker compose up -d

# View logs
docker compose logs -f

# Stop and remove containers, networks
docker compose down

# Stop and also remove volumes
docker compose down -v
```

Compose automatically creates a network so services can reach each other by name (`web` talks to `db` here).

---

## Best Practices

- **Use small base images** like `-slim` or `alpine` variants to reduce size and attack surface.
- **Pin image tags** instead of relying on `latest` for reproducible builds.
- **Add a `.dockerignore`** file to keep build context small (exclude `.git`, `node_modules`, etc.).
- **Order Dockerfile layers** from least to most frequently changing to maximize cache hits.
- **Run as a non-root user** where possible for security.
- **One process per container** — split services rather than cramming them into one image.
- **Don't store secrets in images.** Use environment variables or a secrets manager.

Example `.dockerignore`:

```
.git
node_modules
*.log
.env
__pycache__
```

---

## Common Commands Cheat Sheet

```bash
# Images
docker build -t name:tag .      # Build an image
docker images                   # List images
docker rmi name:tag             # Remove an image
docker pull name:tag            # Download an image

# Containers
docker run -d -p 8080:80 name   # Run detached with port mapping
docker ps -a                    # List all containers
docker exec -it name bash       # Shell into a container
docker stop / start / rm name   # Lifecycle control
docker logs -f name             # Follow logs

# Compose
docker compose up -d            # Start the stack
docker compose down             # Tear it down

# Cleanup
docker system prune -a          # Remove unused data (careful!)
```

---

## Troubleshooting

| Problem | Fix |
|---------|-----|
| `port is already allocated` | Another process uses that host port. Change `-p 8081:80` or free the port. |
| `Cannot connect to the Docker daemon` | Docker Desktop or the Docker service isn't running. Start it. |
| Container exits immediately | Check `docker logs <name>`. The main process likely crashed or finished. |
| Changes not showing up | Rebuild the image (`docker build`) or use a bind mount for live code. |
| Running low on disk | Run `docker system prune -a` to clear unused images, containers, and networks. |

---

## Next Steps

- Containerize one of your own projects with a Dockerfile.
- Convert a multi-service app to Docker Compose.
- Explore multi-stage builds to shrink production images.
- Learn about orchestration with Kubernetes once you're comfortable.

Refer to the [official Docker documentation](https://docs.docker.com) for deeper dives.
