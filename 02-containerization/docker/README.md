# Docker - Containerization Platform

## Table of Contents

- [1. Introduction to Docker](#1-introduction-to-docker)
- [2. Installation Steps](#2-installation-steps)
- [3. Docker Core Concepts](#3-docker-core-concepts)
- [4. Creating a Sample Python Application](#4-creating-a-sample-python-application)
- [5. Writing the Dockerfile](#5-writing-the-dockerfile)
- [6. Building the Docker Image](#6-building-the-docker-image)
- [7. Running the Docker Container Locally](#7-running-the-docker-container-locally)
- [8. Pushing Docker Image to a VM](#8-pushing-docker-image-to-a-vm)
- [9. Essential Docker Commands Reference](#9-essential-docker-commands-reference)
- [10. Docker Best Practices](#10-docker-best-practices)
- [11. Practice Exercises](#11-practice-exercises)

---

## 1. Introduction to Docker

### What Is Docker?

Docker is an open-source platform that automates the deployment, scaling, and management of applications inside lightweight, portable containers. A container packages an application together with all of its dependencies -- libraries, runtime, system tools, and configuration files -- so that it runs identically on any machine that has Docker installed.

Before Docker, developers would constantly run into the "it works on my machine" problem. An application that ran perfectly on a developer's laptop would break in staging or production because of differences in operating system versions, installed libraries, or environment variables. Docker eliminates this class of problem entirely by making the environment itself portable.

### Why Docker Matters in DevOps

Docker is foundational to modern DevOps for several reasons:

- **Consistency**: The same container image runs the same way in development, testing, staging, and production. There is no ambiguity about what version of Python or which system libraries are installed.
- **Speed**: Containers start in seconds, not minutes. This makes development feedback loops faster, CI/CD pipelines quicker, and scaling almost instantaneous.
- **Isolation**: Each container is an isolated process with its own filesystem, networking, and process space. One misbehaving application cannot take down another.
- **Resource Efficiency**: Containers share the host kernel, so they use far less memory and CPU than virtual machines running the same workloads.
- **Microservices Architecture**: Docker makes it practical to break a monolith into small, independently deployable services.
- **Infrastructure as Code**: A Dockerfile is a versioned, reviewable, reproducible definition of your application environment. It lives alongside your code in Git.
- **CI/CD Integration**: Every major CI/CD platform (Jenkins, GitHub Actions, GitLab CI) has first-class Docker support. You build an image once and promote the same artifact through your pipeline.

### Docker vs Virtual Machines

| Aspect | Docker Containers | Virtual Machines |
|---|---|---|
| **Startup time** | Seconds | Minutes |
| **Size** | Megabytes (typically 50-500 MB) | Gigabytes (typically 1-20 GB) |
| **OS** | Shares host kernel | Runs its own full OS kernel |
| **Isolation** | Process-level isolation via namespaces and cgroups | Hardware-level isolation via hypervisor |
| **Performance** | Near-native (no hypervisor overhead) | Slight overhead from hypervisor |
| **Portability** | Highly portable across any Docker host | Portable but heavier to move |
| **Resource usage** | Lightweight, can run dozens on a laptop | Heavy, each VM consumes dedicated RAM/CPU |
| **Use case** | Application packaging and microservices | Running different OS kernels, strong security isolation |

Docker containers are not a replacement for VMs in every scenario. When you need to run a completely different operating system kernel (such as running Windows workloads on a Linux host), or when you need the stronger security boundary that hardware-level isolation provides, VMs are the right choice. For application packaging and deployment, containers are almost always superior.

### Docker Architecture

Docker uses a client-server architecture with the following components:

```
+------------------+       +------------------+       +------------------+
|   Docker Client  | ----> |   Docker Daemon  | ----> |  Docker Registry |
|   (docker CLI)   |  API  |   (dockerd)      |       |  (Docker Hub,    |
|                  |       |                  |       |   private, etc.) |
+------------------+       +------------------+       +------------------+
                                   |
                           +-------+-------+
                           |       |       |
                        Images  Containers  Networks
                                           Volumes
```

- **Docker Client (`docker`)**: The command-line tool you interact with. When you run `docker build` or `docker run`, the client sends these commands to the Docker daemon via a REST API.
- **Docker Daemon (`dockerd`)**: The background service that manages Docker objects (images, containers, networks, volumes). It listens for API requests from the client and handles the heavy lifting of building, running, and distributing containers.
- **Docker Engine**: The combination of the daemon, the REST API, and the CLI. This is what people mean when they say "install Docker."
- **Docker Registry**: A storage and distribution service for Docker images. Docker Hub is the default public registry. Organizations typically also run private registries for their internal images.
- **Docker Objects**: Images (read-only templates), containers (running instances of images), volumes (persistent storage), and networks (communication channels between containers).

---

## 2. Installation Steps

### macOS

#### Option A: Docker Desktop (Recommended)

Docker Desktop is the official way to run Docker on macOS. It includes the Docker daemon, CLI, Docker Compose, and a graphical interface.

1. Download Docker Desktop from [https://www.docker.com/products/docker-desktop](https://www.docker.com/products/docker-desktop)
2. Open the `.dmg` file and drag Docker to your Applications folder
3. Launch Docker from Applications
4. Follow the setup wizard and grant the required permissions
5. Docker Desktop will start the Docker daemon automatically

#### Option B: Homebrew

```bash
# Install Docker Desktop via Homebrew Cask
brew install --cask docker

# Launch Docker Desktop (required -- the daemon does not start automatically)
open /Applications/Docker.app

# Wait for Docker Desktop to finish starting, then verify
docker --version
docker run hello-world
```

#### Option C: Docker CLI only via Homebrew (without Docker Desktop)

If you prefer to run Docker without the Desktop GUI (for example, in a CI environment or if you prefer Colima as a container runtime):

```bash
# Install just the Docker CLI and Docker Compose plugin
brew install docker docker-compose

# Install Colima as the container runtime (replaces Docker Desktop's VM)
brew install colima

# Start Colima (this starts a lightweight Linux VM that runs the Docker daemon)
colima start

# Verify
docker --version
docker run hello-world
```

### Ubuntu/Debian Linux

These are the full, official steps for installing Docker Engine on Ubuntu. Do not install the `docker.io` package from the default Ubuntu repositories -- it is outdated.

```bash
# Step 1: Remove any old or conflicting Docker packages
sudo apt-get remove -y docker docker-engine docker.io containerd runc

# Step 2: Update the package index and install prerequisites
sudo apt-get update
sudo apt-get install -y \
    ca-certificates \
    curl \
    gnupg \
    lsb-release

# Step 3: Add Docker's official GPG key
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | \
    sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

# Step 4: Add the Docker repository to apt sources
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
  https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# Step 5: Install Docker Engine, CLI, and plugins
sudo apt-get update
sudo apt-get install -y \
    docker-ce \
    docker-ce-cli \
    containerd.io \
    docker-buildx-plugin \
    docker-compose-plugin

# Step 6: Verify the installation
sudo docker --version
sudo docker run hello-world
```

For **Debian**, the process is identical except you replace the GPG key URL and repository URL:

```bash
# GPG key for Debian
curl -fsSL https://download.docker.com/linux/debian/gpg | \
    sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg

# Repository for Debian
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
  https://download.docker.com/linux/debian \
  $(lsb_release -cs) stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

### CentOS/RHEL/Fedora

#### CentOS / RHEL

```bash
# Step 1: Remove old Docker packages
sudo yum remove -y docker \
    docker-client \
    docker-client-latest \
    docker-common \
    docker-latest \
    docker-latest-logrotate \
    docker-logrotate \
    docker-engine

# Step 2: Install required utilities
sudo yum install -y yum-utils

# Step 3: Add the Docker repository
sudo yum-config-manager --add-repo \
    https://download.docker.com/linux/centos/docker-ce.repo

# Step 4: Install Docker Engine
sudo yum install -y \
    docker-ce \
    docker-ce-cli \
    containerd.io \
    docker-buildx-plugin \
    docker-compose-plugin

# Step 5: Start and enable Docker
sudo systemctl start docker
sudo systemctl enable docker

# Step 6: Verify
sudo docker --version
sudo docker run hello-world
```

#### Fedora

```bash
# Step 1: Remove old Docker packages
sudo dnf remove -y docker \
    docker-client \
    docker-client-latest \
    docker-common \
    docker-latest \
    docker-latest-logrotate \
    docker-logrotate \
    docker-selinux \
    docker-engine-selinux \
    docker-engine

# Step 2: Install required utilities
sudo dnf install -y dnf-plugins-core

# Step 3: Add the Docker repository
sudo dnf config-manager --add-repo \
    https://download.docker.com/linux/fedora/docker-ce.repo

# Step 4: Install Docker Engine
sudo dnf install -y \
    docker-ce \
    docker-ce-cli \
    containerd.io \
    docker-buildx-plugin \
    docker-compose-plugin

# Step 5: Start and enable Docker
sudo systemctl start docker
sudo systemctl enable docker

# Step 6: Verify
sudo docker --version
sudo docker run hello-world
```

### Windows

#### Docker Desktop for Windows

1. **System Requirements**: Windows 10/11 64-bit with WSL 2 backend (recommended) or Hyper-V
2. Download Docker Desktop from [https://www.docker.com/products/docker-desktop](https://www.docker.com/products/docker-desktop)
3. Run the installer
4. When prompted, ensure **"Use WSL 2 instead of Hyper-V"** is checked (recommended)
5. Restart your computer if prompted
6. Launch Docker Desktop from the Start menu

If WSL 2 is not yet installed:

```powershell
# Open PowerShell as Administrator
wsl --install

# Restart your computer, then verify
wsl --version
```

After Docker Desktop is running, open a terminal (PowerShell, Command Prompt, or WSL) and verify:

```powershell
docker --version
docker run hello-world
```

### Post-Installation Steps (Linux)

On Linux, the Docker daemon runs as root. By default, you need `sudo` for every `docker` command. To run Docker as a non-root user:

```bash
# Create the docker group (usually already exists after installation)
sudo groupadd docker

# Add your user to the docker group
sudo usermod -aG docker $USER

# Activate the new group membership without logging out
newgrp docker

# Verify you can run docker without sudo
docker run hello-world
```

**Why this matters**: In a development environment, typing `sudo` before every Docker command is tedious. In production, you should carefully control who is in the `docker` group because membership effectively grants root-equivalent access to the host.

Configure Docker to start on boot:

```bash
# Enable Docker to start on boot
sudo systemctl enable docker
sudo systemctl enable containerd

# Check Docker daemon status
sudo systemctl status docker
```

### Docker Compose Installation

If you installed Docker Engine using the steps above (with `docker-compose-plugin`), Docker Compose is already available as `docker compose` (note: no hyphen). Verify:

```bash
docker compose version
```

If you need the standalone `docker-compose` binary (legacy, for older scripts):

```bash
# Download the latest stable release
sudo curl -L "https://github.com/docker/compose/releases/latest/download/docker-compose-$(uname -s)-$(uname -m)" \
    -o /usr/local/bin/docker-compose

# Make it executable
sudo chmod +x /usr/local/bin/docker-compose

# Verify
docker-compose --version
```

### Verifying Your Full Installation

Run these commands to confirm everything is working:

```bash
# Docker version (client and server)
docker version

# Detailed system information
docker info

# Run a test container
docker run hello-world

# Check Docker Compose
docker compose version
```

---

## 3. Docker Core Concepts

### Images

A Docker image is a read-only template that contains a set of instructions for creating a container. Think of it as a snapshot of a filesystem plus metadata (what command to run, which port to expose, etc.).

Key characteristics:
- **Layered**: Images are built from a series of layers. Each instruction in a Dockerfile creates a new layer. Layers are cached and shared across images, saving disk space and build time.
- **Immutable**: Once built, an image does not change. You create a new image (with a new tag) when you need changes.
- **Portable**: An image built on your laptop will run identically on any machine with Docker installed.

```bash
# List images on your machine
docker images

# Pull an image from Docker Hub
docker pull python:3.12-slim

# Inspect image details
docker inspect python:3.12-slim

# View the layers of an image
docker history python:3.12-slim

# Remove an image
docker rmi python:3.12-slim
```

### Containers

A container is a running instance of an image. If an image is a class, a container is an object. You can create multiple containers from the same image, each running independently.

Key characteristics:
- **Ephemeral**: Containers are designed to be disposable. Stop one, start a new one from the same image, and you are back to a known-good state.
- **Writable layer**: Each container gets a thin writable layer on top of the image's read-only layers. Changes made inside the container (new files, modified files) exist only in this writable layer and are lost when the container is removed -- unless you use volumes.
- **Isolated**: Each container has its own filesystem, network interface, and process tree. Containers cannot see each other's processes by default.

```bash
# List running containers
docker ps

# List all containers (including stopped ones)
docker ps -a

# Start a stopped container
docker start <container_id>

# Stop a running container
docker stop <container_id>

# Remove a container
docker rm <container_id>

# Remove all stopped containers
docker container prune
```

### Volumes

Volumes are Docker's mechanism for persisting data generated by and used by containers. Without volumes, all data inside a container is lost when the container is removed.

There are three types of mounts:
- **Named volumes**: Managed by Docker, stored in `/var/lib/docker/volumes/`. Best for most use cases.
- **Bind mounts**: Maps a specific host directory to a container directory. Useful for development (mount your source code into the container).
- **tmpfs mounts**: Stored in host memory only, never written to disk. Useful for sensitive data that should not persist.

```bash
# Create a named volume
docker volume create my-data

# List volumes
docker volume ls

# Inspect a volume
docker volume inspect my-data

# Run a container with a named volume
docker run -v my-data:/app/data my-image

# Run a container with a bind mount (maps host directory into container)
docker run -v $(pwd)/src:/app/src my-image

# Remove a volume
docker volume rm my-data

# Remove all unused volumes
docker volume prune
```

### Networks

Docker networking allows containers to communicate with each other and with the outside world.

Docker provides several network drivers:
- **bridge** (default): Containers on the same bridge network can communicate by container name. This is the most common network type.
- **host**: The container shares the host's network stack. No port mapping is needed, but you lose isolation. (Linux only.)
- **none**: No networking. The container is completely isolated.
- **overlay**: For multi-host networking in Docker Swarm or Kubernetes.

```bash
# List networks
docker network ls

# Create a custom bridge network
docker network create my-network

# Run containers on the same network (they can reach each other by name)
docker run -d --name app --network my-network my-app
docker run -d --name db --network my-network postgres:16

# Inside the "app" container, you can connect to the database at hostname "db"

# Inspect a network
docker network inspect my-network

# Remove a network
docker network rm my-network
```

### Dockerfile Syntax Overview

A Dockerfile is a text file containing instructions for building a Docker image. Each instruction creates a layer.

| Instruction | Purpose | Example |
|---|---|---|
| `FROM` | Set the base image | `FROM python:3.12-slim` |
| `WORKDIR` | Set the working directory inside the container | `WORKDIR /app` |
| `COPY` | Copy files from host to container | `COPY . /app` |
| `ADD` | Like COPY but can also extract archives and fetch URLs | `ADD archive.tar.gz /app` |
| `RUN` | Execute a command during the build (creates a new layer) | `RUN pip install flask` |
| `ENV` | Set an environment variable | `ENV FLASK_ENV=production` |
| `EXPOSE` | Document which port the container listens on | `EXPOSE 5000` |
| `CMD` | Default command when the container starts | `CMD ["python", "app.py"]` |
| `ENTRYPOINT` | Like CMD but harder to override; sets the executable | `ENTRYPOINT ["python"]` |
| `ARG` | Build-time variable (not available at runtime) | `ARG VERSION=1.0` |
| `LABEL` | Add metadata to the image | `LABEL maintainer="you@example.com"` |
| `VOLUME` | Create a mount point for a volume | `VOLUME /app/data` |
| `USER` | Set the user for subsequent instructions and the container | `USER appuser` |
| `HEALTHCHECK` | Define a command to check container health | `HEALTHCHECK CMD curl -f http://localhost:5000/health` |
| `STOPSIGNAL` | Set the signal to stop the container | `STOPSIGNAL SIGTERM` |

### Docker Hub and Registries

A Docker registry is a service that stores and distributes Docker images.

- **Docker Hub** ([https://hub.docker.com](https://hub.docker.com)): The default public registry. Contains official images (like `python`, `nginx`, `postgres`) and community images.
- **GitHub Container Registry** (`ghcr.io`): GitHub's container registry, tied to your GitHub repositories.
- **Amazon ECR**: AWS's private container registry.
- **Google Artifact Registry**: GCP's container registry.
- **Azure Container Registry**: Azure's container registry.
- **Self-hosted registry**: You can run your own registry using the `registry:2` Docker image.

```bash
# Search for images on Docker Hub
docker search python

# Pull an official image
docker pull python:3.12-slim

# Pull from a specific registry
docker pull ghcr.io/owner/image:tag
```

---

## 4. Creating a Sample Python Application

Before containerizing anything, we need an application. We will build a simple Flask REST API with two endpoints: a health check and a greeting endpoint. This is deliberately simple so the focus remains on Docker, not on Flask.

### Project Structure

```
flask-docker-app/
    app.py
    requirements.txt
    Dockerfile
    .dockerignore
    docker-compose.yml
```

### Step 1: Create the Application Directory

```bash
mkdir -p flask-docker-app
cd flask-docker-app
```

### Step 2: Create `requirements.txt`

This file lists the Python dependencies for the application.

```txt
Flask==3.1.0
gunicorn==23.0.0
```

**Why gunicorn?** Flask's built-in development server is single-threaded and not suitable for production. Gunicorn is a production-grade WSGI HTTP server that can handle multiple concurrent requests.

### Step 3: Create `app.py`

```python
"""
A simple Flask REST API for demonstrating Docker containerization.

Endpoints:
    GET /health  - Health check endpoint (used by Docker HEALTHCHECK)
    GET /        - Welcome message
    GET /hello   - Returns a greeting, optionally personalized with ?name=
"""

import os
from flask import Flask, jsonify, request

app = Flask(__name__)

# Read configuration from environment variables (a Docker best practice)
APP_PORT = int(os.environ.get("APP_PORT", 5000))
APP_ENV = os.environ.get("APP_ENV", "development")


@app.route("/health")
def health():
    """Health check endpoint.

    Returns a simple JSON response indicating the service is running.
    This endpoint is used by Docker's HEALTHCHECK instruction and by
    load balancers to determine if the container is healthy.
    """
    return jsonify({
        "status": "healthy",
        "environment": APP_ENV
    }), 200


@app.route("/")
def index():
    """Root endpoint returning a welcome message."""
    return jsonify({
        "message": "Welcome to the Flask Docker Demo API",
        "version": "1.0.0",
        "endpoints": {
            "/": "This welcome message",
            "/health": "Health check",
            "/hello": "Greeting endpoint (use ?name=YourName)"
        }
    }), 200


@app.route("/hello")
def hello():
    """Greeting endpoint.

    Accepts an optional 'name' query parameter.
    Examples:
        GET /hello          -> {"greeting": "Hello, World!"}
        GET /hello?name=Ada -> {"greeting": "Hello, Ada!"}
    """
    name = request.args.get("name", "World")
    return jsonify({
        "greeting": f"Hello, {name}!"
    }), 200


if __name__ == "__main__":
    # This block runs only during local development (python app.py).
    # In production, gunicorn runs the app directly and this block is skipped.
    print(f"Starting Flask app in {APP_ENV} mode on port {APP_PORT}")
    app.run(host="0.0.0.0", port=APP_PORT, debug=(APP_ENV == "development"))
```

### Step 4: Run the Application Locally (Without Docker)

Before putting anything into a container, always make sure it works locally. This makes debugging much easier.

```bash
# Create a virtual environment
python3 -m venv venv
source venv/bin/activate

# Install dependencies
pip install -r requirements.txt

# Run the application
python app.py
```

The application will start and print:

```
Starting Flask app in development mode on port 5000
 * Running on http://0.0.0.0:5000
```

### Step 5: Test the Application with curl

Open a new terminal window and run:

```bash
# Test the root endpoint
curl http://localhost:5000/
# Expected: {"endpoints":{...},"message":"Welcome to the Flask Docker Demo API","version":"1.0.0"}

# Test the health endpoint
curl http://localhost:5000/health
# Expected: {"environment":"development","status":"healthy"}

# Test the hello endpoint without a name
curl http://localhost:5000/hello
# Expected: {"greeting":"Hello, World!"}

# Test the hello endpoint with a name
curl http://localhost:5000/hello?name=DevOps
# Expected: {"greeting":"Hello, DevOps!"}
```

Once everything works, stop the application with `Ctrl+C` and deactivate the virtual environment:

```bash
deactivate
```

---

## 5. Writing the Dockerfile

### Simple Single-Stage Dockerfile (For Learning)

This is the easiest Dockerfile to understand. It is fine for learning and development, but we will improve it for production afterward.

```dockerfile
# Use the official Python 3.12 slim image as the base.
# "slim" is a smaller variant of the full Python image (roughly 150 MB vs 1 GB).
FROM python:3.12-slim

# Set the working directory inside the container.
# All subsequent commands (COPY, RUN, CMD) will execute relative to /app.
WORKDIR /app

# Copy the requirements file first, then install dependencies.
# This is done BEFORE copying the rest of the code so that Docker can
# cache the dependency installation layer. If requirements.txt has not
# changed, Docker reuses the cached layer and skips pip install.
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Now copy the rest of the application code.
COPY . .

# Document that the container listens on port 5000.
# This does NOT actually publish the port -- that is done with docker run -p.
EXPOSE 5000

# Set environment variables.
ENV APP_ENV=production
ENV APP_PORT=5000

# The default command to run when the container starts.
# Using gunicorn instead of the Flask development server.
CMD ["gunicorn", "--bind", "0.0.0.0:5000", "--workers", "2", "app:app"]
```

### Production-Ready Multi-Stage Dockerfile

A multi-stage build uses multiple `FROM` instructions. Each `FROM` starts a new build stage. You can selectively copy artifacts from one stage to another. This is useful for keeping the final image small -- build tools and intermediate files stay in earlier stages and never appear in the final image.

For a Python application, the multi-stage approach is especially valuable when you have packages that require compilation (like `psycopg2` or `numpy`). The build stage contains the compiler and development headers; the final stage contains only the compiled wheels.

```dockerfile
# ==============================================================================
# Stage 1: Builder
# ==============================================================================
# This stage installs dependencies. Any build tools needed for compilation
# exist only in this stage and do not bloat the final image.
FROM python:3.12-slim AS builder

WORKDIR /app

# Install system-level build dependencies if needed (example: for packages
# that compile C extensions). Uncomment the following if your requirements
# include packages like psycopg2, numpy, etc.
# RUN apt-get update && apt-get install -y --no-install-recommends \
#     gcc \
#     libpq-dev \
#     && rm -rf /var/lib/apt/lists/*

# Copy and install Python dependencies into a virtual environment.
# Using a venv makes it easy to copy all installed packages to the final stage.
RUN python -m venv /opt/venv
ENV PATH="/opt/venv/bin:$PATH"

COPY requirements.txt .
RUN pip install --no-cache-dir --upgrade pip && \
    pip install --no-cache-dir -r requirements.txt

# ==============================================================================
# Stage 2: Production
# ==============================================================================
# Start from a clean slim image. Only copy the venv and application code.
FROM python:3.12-slim AS production

# Add metadata labels
LABEL maintainer="your-email@example.com"
LABEL description="Flask Docker Demo API"
LABEL version="1.0.0"

# Create a non-root user for security.
# Running as root inside a container is a security risk because if an attacker
# escapes the container, they have root access on the host.
RUN groupadd --gid 1000 appuser && \
    useradd --uid 1000 --gid 1000 --create-home --shell /bin/bash appuser

WORKDIR /app

# Copy the virtual environment from the builder stage
COPY --from=builder /opt/venv /opt/venv
ENV PATH="/opt/venv/bin:$PATH"

# Copy application code
COPY --chown=appuser:appuser . .

# Set environment variables
ENV APP_ENV=production
ENV APP_PORT=5000
ENV PYTHONUNBUFFERED=1
ENV PYTHONDONTWRITEBYTECODE=1

# Expose the application port
EXPOSE 5000

# Define a health check so Docker can monitor the container
HEALTHCHECK --interval=30s --timeout=5s --start-period=10s --retries=3 \
    CMD curl -f http://localhost:5000/health || exit 1

# Switch to the non-root user
USER appuser

# Run the application with gunicorn
CMD ["gunicorn", "--bind", "0.0.0.0:5000", "--workers", "2", "--access-logfile", "-", "app:app"]
```

### Detailed Explanation of Every Instruction

**`FROM python:3.12-slim AS builder`**: Sets the base image. The `AS builder` gives this stage a name so we can reference it later with `COPY --from=builder`. The `slim` variant contains only the minimal packages needed to run Python, reducing image size significantly compared to the full `python:3.12` image.

**`WORKDIR /app`**: Sets the working directory for all subsequent commands. If the directory does not exist, Docker creates it. This is preferred over `RUN mkdir /app && cd /app` because WORKDIR persists across instructions.

**`RUN python -m venv /opt/venv`**: Creates a Python virtual environment. This isolates our dependencies and makes it trivial to copy them to the next stage.

**`ENV PATH="/opt/venv/bin:$PATH"`**: Puts the virtual environment's `bin` directory at the front of PATH so `pip` and `python` resolve to the venv copies.

**`COPY requirements.txt .`**: Copies only the requirements file. We do this before `COPY . .` to take advantage of Docker's build cache. If `requirements.txt` has not changed since the last build, Docker skips the `pip install` step entirely.

**`RUN pip install --no-cache-dir -r requirements.txt`**: Installs dependencies. The `--no-cache-dir` flag tells pip not to store downloaded packages in a cache directory, reducing the image size.

**`FROM python:3.12-slim AS production`**: Starts a new, clean build stage. Nothing from the `builder` stage exists here unless we explicitly copy it.

**`RUN groupadd ... && useradd ...`**: Creates a non-root user. Running the application as a non-root user is a critical security practice.

**`COPY --from=builder /opt/venv /opt/venv`**: Copies the installed dependencies from the builder stage into the production stage. Build tools, temporary files, and the pip cache from the builder stage are left behind.

**`COPY --chown=appuser:appuser . .`**: Copies the application code and sets ownership to the non-root user in a single step.

**`ENV PYTHONUNBUFFERED=1`**: Forces Python to send output directly to stdout/stderr without buffering. This ensures you see logs in real time with `docker logs`.

**`ENV PYTHONDONTWRITEBYTECODE=1`**: Prevents Python from writing `.pyc` files to disk. These files are unnecessary in a container and just take up space.

**`EXPOSE 5000`**: Documents that the application listens on port 5000. This is metadata only -- it does not publish the port. You still need `-p 5000:5000` when running the container.

**`HEALTHCHECK`**: Defines a command that Docker runs periodically to check if the container is healthy. If the health check fails consecutively (based on `--retries`), Docker marks the container as `unhealthy`. Orchestrators like Docker Swarm and Kubernetes use this to restart unhealthy containers.

**`USER appuser`**: Switches to the non-root user for all subsequent instructions and for the container's runtime.

**`CMD ["gunicorn", ...]`**: The default command. The exec form (JSON array) is preferred over the shell form because it runs gunicorn directly as PID 1, meaning it receives signals (like SIGTERM for graceful shutdown) correctly.

### The `.dockerignore` File

Just like `.gitignore` tells Git which files to skip, `.dockerignore` tells Docker which files to exclude from the build context. This is important for two reasons:

1. **Security**: Prevents sensitive files (like `.env` files with secrets, SSH keys) from being copied into the image.
2. **Performance**: Reduces the build context size, making `docker build` faster.

Create a file named `.dockerignore`:

```
# Version control
.git
.gitignore

# Python artifacts
__pycache__
*.pyc
*.pyo
*.egg-info
dist
build
*.egg

# Virtual environment
venv
.venv
env

# IDE and editor files
.vscode
.idea
*.swp
*.swo
*~

# OS files
.DS_Store
Thumbs.db

# Docker files (prevent recursive builds)
Dockerfile
docker-compose.yml
docker-compose*.yml

# Environment files with secrets
.env
.env.*

# Documentation
README.md
LICENSE
docs/

# Testing
tests/
.pytest_cache
.coverage
htmlcov
```

### Best Practices for Dockerfiles

1. **Order layers from least to most frequently changing**. Base image and dependency installation rarely change. Application code changes frequently. By putting `COPY requirements.txt` and `RUN pip install` before `COPY . .`, you ensure that changing your code does not invalidate the dependency installation cache.

2. **Use specific image tags, not `latest`**. `FROM python:3.12-slim` is reproducible. `FROM python:latest` could break your build tomorrow when a new Python version is released.

3. **Minimize the number of layers**. Combine related `RUN` commands with `&&`:
   ```dockerfile
   # Good: single layer
   RUN apt-get update && apt-get install -y curl && rm -rf /var/lib/apt/lists/*

   # Bad: three layers, and the apt cache from the first layer persists
   RUN apt-get update
   RUN apt-get install -y curl
   RUN rm -rf /var/lib/apt/lists/*
   ```

4. **Use multi-stage builds** to keep final images small.

5. **Run as a non-root user** for security.

6. **Use `.dockerignore`** to keep the build context small and prevent secrets from leaking into images.

7. **Use `COPY` instead of `ADD`** unless you specifically need ADD's ability to extract archives or fetch URLs.

8. **Set `HEALTHCHECK`** so orchestrators know whether your container is actually working.

---

## 6. Building the Docker Image

### Basic Build Command

Navigate to the directory containing your Dockerfile and run:

```bash
# Build the image and tag it
docker build -t flask-docker-app:1.0.0 .
```

Breaking this down:
- `docker build`: The build command
- `-t flask-docker-app:1.0.0`: Tags the image with a name (`flask-docker-app`) and a version (`1.0.0`)
- `.`: The build context -- the directory containing the Dockerfile and application files. Docker sends this entire directory (minus `.dockerignore` exclusions) to the daemon.

### Useful Build Flags

```bash
# Build with a specific Dockerfile (if not named "Dockerfile")
docker build -f Dockerfile.prod -t flask-docker-app:1.0.0 .

# Build without using cache (force a fresh build)
docker build --no-cache -t flask-docker-app:1.0.0 .

# Build a specific stage in a multi-stage Dockerfile
docker build --target builder -t flask-docker-app:builder .

# Build with build arguments
docker build --build-arg VERSION=2.0.0 -t flask-docker-app:2.0.0 .

# Build with multiple tags at once
docker build -t flask-docker-app:1.0.0 -t flask-docker-app:latest .

# Build for a specific platform (useful for cross-platform builds)
docker build --platform linux/amd64 -t flask-docker-app:1.0.0 .

# Build and show full output (not collapsed)
docker build --progress=plain -t flask-docker-app:1.0.0 .
```

### Tagging Strategies

Tags are how you identify different versions of an image. A well-thought-out tagging strategy is essential for production deployments.

```bash
# Tag an existing image with a new tag
docker tag flask-docker-app:1.0.0 flask-docker-app:latest

# Tag for pushing to Docker Hub (must include your Docker Hub username)
docker tag flask-docker-app:1.0.0 yourusername/flask-docker-app:1.0.0

# Tag for pushing to a private registry
docker tag flask-docker-app:1.0.0 registry.example.com/flask-docker-app:1.0.0
```

Common tagging conventions:
- **Semantic versioning**: `1.0.0`, `1.0.1`, `1.1.0`, `2.0.0`
- **Git commit SHA**: `flask-docker-app:a1b2c3d` -- unambiguously identifies which code is in the image
- **Git branch + SHA**: `flask-docker-app:main-a1b2c3d`
- **latest**: Points to the most recent stable build. Be cautious with `latest` in production -- it is mutable and can change unexpectedly.
- **Environment tags**: `flask-docker-app:staging`, `flask-docker-app:production`

A robust CI/CD pipeline typically tags each image with both the semantic version and the Git SHA:

```bash
docker build \
    -t flask-docker-app:1.0.0 \
    -t flask-docker-app:$(git rev-parse --short HEAD) \
    -t flask-docker-app:latest \
    .
```

### Viewing Images and Image Details

```bash
# List all local images
docker images

# List images with filtering
docker images flask-docker-app

# Show image size
docker images --format "table {{.Repository}}\t{{.Tag}}\t{{.Size}}"

# Inspect detailed image metadata (labels, layers, config)
docker inspect flask-docker-app:1.0.0

# View the layers that make up an image and their sizes
docker history flask-docker-app:1.0.0

# View layers with full commands (not truncated)
docker history --no-trunc flask-docker-app:1.0.0
```

### Image Size Optimization

Image size directly affects:
- **Build time**: Larger images take longer to build.
- **Push/pull time**: Larger images take longer to upload and download.
- **Startup time**: Larger images take longer to pull on a new host.
- **Attack surface**: Larger images contain more packages, which means more potential vulnerabilities.

Strategies for reducing image size:

```bash
# Check the size of your image
docker images flask-docker-app

# Compare base image sizes
docker pull python:3.12          # ~1 GB (full Debian with dev tools)
docker pull python:3.12-slim     # ~150 MB (minimal Debian)
docker pull python:3.12-alpine   # ~50 MB (Alpine Linux, but can cause compatibility issues)

docker images python
```

| Base Image | Size | Notes |
|---|---|---|
| `python:3.12` | ~1 GB | Full Debian, includes gcc and build tools |
| `python:3.12-slim` | ~150 MB | Minimal Debian, good balance of size and compatibility |
| `python:3.12-alpine` | ~50 MB | Smallest, but uses musl libc which can break some Python packages |

Recommendation: Use `slim` for most applications. Use `alpine` only if you have tested thoroughly and do not need packages that depend on glibc.

### Build Cache and How It Works

Docker caches each layer (each instruction in the Dockerfile). When you rebuild, Docker checks whether the inputs for each instruction have changed:

1. `FROM python:3.12-slim` -- cached if the base image has not been updated
2. `COPY requirements.txt .` -- cached if the file contents have not changed
3. `RUN pip install ...` -- cached if the previous layer was cached (since the requirements file is the same)
4. `COPY . .` -- invalidated if ANY file in the build context has changed

Once a cache is invalidated, every subsequent layer must also be rebuilt. This is why the order of instructions matters so much.

```bash
# First build (no cache, everything is built from scratch)
docker build -t flask-docker-app:1.0.0 .
# Output: each step shows "Running" or "Downloading"

# Second build with no code changes (fully cached, takes seconds)
docker build -t flask-docker-app:1.0.0 .
# Output: each step shows "CACHED"

# Third build after changing app.py but not requirements.txt
# The pip install layer is still cached -- only COPY . . and later are rebuilt
docker build -t flask-docker-app:1.0.0 .

# Force a completely fresh build (ignore all cache)
docker build --no-cache -t flask-docker-app:1.0.0 .
```

### Removing Unused Images

Over time, old images accumulate. Clean them up:

```bash
# Remove a specific image
docker rmi flask-docker-app:1.0.0

# Remove all dangling images (layers not referenced by any tagged image)
docker image prune

# Remove ALL unused images (not just dangling ones)
docker image prune -a

# Nuclear option: remove everything (images, containers, volumes, networks)
docker system prune -a --volumes
```

---

## 7. Running the Docker Container Locally

### Basic Run Command

```bash
docker run -d -p 5000:5000 --name flask-app flask-docker-app:1.0.0
```

Breaking this down:
- `docker run`: Creates and starts a new container from an image
- `-d`: Detached mode -- runs the container in the background
- `-p 5000:5000`: Maps port 5000 on the host to port 5000 in the container. Format: `host_port:container_port`
- `--name flask-app`: Gives the container a human-readable name instead of a random one
- `flask-docker-app:1.0.0`: The image to use

### All Important `docker run` Flags

```bash
# Run in the foreground (attached mode, see output directly)
docker run -p 5000:5000 flask-docker-app:1.0.0

# Run in detached mode (background)
docker run -d -p 5000:5000 flask-docker-app:1.0.0

# Automatically remove the container when it stops
docker run --rm -p 5000:5000 flask-docker-app:1.0.0

# Set environment variables
docker run -d -p 5000:5000 \
    -e APP_ENV=staging \
    -e APP_PORT=5000 \
    --name flask-app \
    flask-docker-app:1.0.0

# Load environment variables from a file
# (Create a file called .env with KEY=VALUE pairs, one per line)
docker run -d -p 5000:5000 --env-file .env --name flask-app flask-docker-app:1.0.0

# Mount a volume (persist data)
docker run -d -p 5000:5000 \
    -v app-data:/app/data \
    --name flask-app \
    flask-docker-app:1.0.0

# Bind mount (map a host directory into the container -- great for development)
docker run -d -p 5000:5000 \
    -v $(pwd):/app \
    --name flask-app \
    flask-docker-app:1.0.0

# Connect to a specific network
docker run -d -p 5000:5000 \
    --network my-network \
    --name flask-app \
    flask-docker-app:1.0.0

# Set resource limits
docker run -d -p 5000:5000 \
    --memory="256m" \
    --cpus="0.5" \
    --name flask-app \
    flask-docker-app:1.0.0

# Restart policy (restart automatically on failure or on reboot)
docker run -d -p 5000:5000 \
    --restart unless-stopped \
    --name flask-app \
    flask-docker-app:1.0.0

# Override the CMD from the Dockerfile
docker run --rm flask-docker-app:1.0.0 python -c "print('hello')"

# Run an interactive shell inside the container
docker run --rm -it flask-docker-app:1.0.0 /bin/bash
```

### Restart Policies

| Policy | Behavior |
|---|---|
| `no` | Never restart (default) |
| `on-failure` | Restart only if the container exits with a non-zero exit code |
| `on-failure:5` | Same as above, but stop trying after 5 attempts |
| `always` | Always restart, regardless of exit code |
| `unless-stopped` | Like `always`, but does not restart if the container was manually stopped |

For production, `unless-stopped` is typically the right choice.

### Test the Running Container

```bash
# Verify the container is running
docker ps

# Test the endpoints
curl http://localhost:5000/
curl http://localhost:5000/health
curl http://localhost:5000/hello
curl http://localhost:5000/hello?name=Docker
```

### Viewing Logs

```bash
# View all logs from a container
docker logs flask-app

# Follow logs in real time (like tail -f)
docker logs -f flask-app

# Show the last 50 lines
docker logs --tail 50 flask-app

# Show logs with timestamps
docker logs -t flask-app

# Show logs since a specific time
docker logs --since 2024-01-01T00:00:00 flask-app

# Show logs from the last 5 minutes
docker logs --since 5m flask-app
```

### Executing Commands Inside a Running Container

```bash
# Open an interactive shell inside the running container
docker exec -it flask-app /bin/bash

# Run a single command inside the container
docker exec flask-app ls -la /app

# Check Python version inside the container
docker exec flask-app python --version

# Check running processes inside the container
docker exec flask-app ps aux

# Check environment variables
docker exec flask-app env

# Run as a specific user
docker exec -u root flask-app whoami
```

### Stopping and Removing Containers

```bash
# Stop a running container (sends SIGTERM, waits 10 seconds, then SIGKILL)
docker stop flask-app

# Stop with a custom timeout (wait 30 seconds before SIGKILL)
docker stop -t 30 flask-app

# Force kill a container immediately (sends SIGKILL -- use as last resort)
docker kill flask-app

# Remove a stopped container
docker rm flask-app

# Stop and remove in one command
docker rm -f flask-app

# Remove all stopped containers
docker container prune

# Pause a container (freeze all processes -- useful for debugging)
docker pause flask-app

# Unpause
docker unpause flask-app
```

### Docker Compose for Local Development

Docker Compose lets you define and run multi-container applications with a single YAML file. Even for a single container, Compose is convenient because it saves you from typing long `docker run` commands.

Create a file named `docker-compose.yml`:

```yaml
services:
  # ---- Flask Application ----
  flask-app:
    build:
      context: .
      dockerfile: Dockerfile
    container_name: flask-app
    ports:
      - "5000:5000"
    environment:
      - APP_ENV=development
      - APP_PORT=5000
    volumes:
      # Bind mount for live code reloading during development.
      # Changes to app.py on your host are immediately reflected in the container.
      - .:/app
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:5000/health"]
      interval: 30s
      timeout: 5s
      retries: 3
      start_period: 10s
    networks:
      - app-network

  # ---- Redis (example of a second service) ----
  redis:
    image: redis:7-alpine
    container_name: redis
    ports:
      - "6379:6379"
    volumes:
      - redis-data:/data
    restart: unless-stopped
    networks:
      - app-network

# Named volumes persist data across container restarts
volumes:
  redis-data:

# Custom network so containers can communicate by name
networks:
  app-network:
    driver: bridge
```

#### Docker Compose Commands

```bash
# Start all services in the background
docker compose up -d

# Start all services and rebuild images if Dockerfile has changed
docker compose up -d --build

# View running services
docker compose ps

# View logs for all services
docker compose logs

# View logs for a specific service, following in real time
docker compose logs -f flask-app

# Stop all services
docker compose stop

# Stop and remove all containers, networks, and volumes
docker compose down

# Stop, remove everything, AND delete volumes (careful -- destroys data)
docker compose down -v

# Restart a single service
docker compose restart flask-app

# Execute a command in a running service
docker compose exec flask-app /bin/bash

# Scale a service (run multiple instances)
docker compose up -d --scale flask-app=3

# Build images without starting containers
docker compose build

# Pull the latest images for all services
docker compose pull
```

### Health Checks in Detail

The `HEALTHCHECK` instruction (in the Dockerfile) or `healthcheck` key (in docker-compose.yml) defines how Docker determines if your container is healthy.

```bash
# View container health status
docker inspect --format='{{.State.Health.Status}}' flask-app

# View health check log (last few results)
docker inspect --format='{{json .State.Health}}' flask-app | python -m json.tool
```

A container goes through three health states:
1. **starting**: The container has just started and is within the `start_period`. Health checks that fail during this time do not count against the retries.
2. **healthy**: The health check command returned exit code 0.
3. **unhealthy**: The health check command failed `retries` consecutive times.

---

## 8. Pushing Docker Image to a VM

You have built a Docker image on your local machine. Now you need to get it running on a remote VM (a server in the cloud, a staging environment, a production host). There are three methods for doing this, each with its own trade-offs.

### Method A: Using Docker Hub (Public Registry)

This is the simplest approach. You push your image to Docker Hub, then pull it on the VM. Docker Hub is free for public images and offers a free tier for private images.

**When to use**: Quick deployments, open-source projects, learning and experimentation.

#### Step 1: Create a Docker Hub Account

1. Go to [https://hub.docker.com](https://hub.docker.com) and sign up
2. Verify your email address
3. Your username becomes your namespace on Docker Hub (e.g., `yourusername/flask-docker-app`)

#### Step 2: Log In from the CLI

```bash
# Log in to Docker Hub
docker login

# You will be prompted for your username and password (or access token).
# Using an access token is recommended over a password.
# Create a token at: https://hub.docker.com/settings/security

# To log in non-interactively (useful in CI/CD):
echo "$DOCKER_HUB_TOKEN" | docker login --username yourusername --password-stdin
```

#### Step 3: Tag the Image for Docker Hub

Docker Hub expects image names in the format `username/repository:tag`:

```bash
# Tag your local image for Docker Hub
docker tag flask-docker-app:1.0.0 yourusername/flask-docker-app:1.0.0
docker tag flask-docker-app:1.0.0 yourusername/flask-docker-app:latest
```

#### Step 4: Push the Image

```bash
# Push a specific tag
docker push yourusername/flask-docker-app:1.0.0

# Push all tags for this repository
docker push yourusername/flask-docker-app --all-tags
```

You can verify the push by visiting `https://hub.docker.com/r/yourusername/flask-docker-app`.

#### Step 5: On the VM -- Pull and Run

SSH into your VM and run:

```bash
# SSH into the VM
ssh user@your-vm-ip-address

# Install Docker on the VM if not already installed (see Section 2 above for
# full installation steps for your VM's OS)

# Pull the image from Docker Hub
docker pull yourusername/flask-docker-app:1.0.0

# Run the container
docker run -d \
    -p 80:5000 \
    --name flask-app \
    --restart unless-stopped \
    -e APP_ENV=production \
    yourusername/flask-docker-app:1.0.0

# Verify it is running
docker ps
curl http://localhost:80/health
```

Note that we mapped host port 80 (the standard HTTP port) to container port 5000. This means users can access the application at `http://your-vm-ip-address/` without specifying a port.

#### Step 6: Log Out (Security)

```bash
# Always log out when done, especially on shared machines
docker logout
```

### Method B: Direct Transfer Without a Registry

Sometimes you cannot use a registry -- maybe you are in an air-gapped environment, or you do not want to push a proprietary image to a third-party service. In that case, you can export the image as a tar file and transfer it directly.

**When to use**: Air-gapped environments, quick one-off deployments, no registry available.

#### Step 1: Save the Image to a Tar File

```bash
# Save a single image
docker save -o flask-docker-app-1.0.0.tar flask-docker-app:1.0.0

# Check the file size
ls -lh flask-docker-app-1.0.0.tar

# Save with gzip compression (significantly smaller file)
docker save flask-docker-app:1.0.0 | gzip > flask-docker-app-1.0.0.tar.gz

# Check compressed size
ls -lh flask-docker-app-1.0.0.tar.gz
```

#### Step 2: Transfer the Tar File to the VM

```bash
# Using scp (Secure Copy Protocol)
scp flask-docker-app-1.0.0.tar.gz user@your-vm-ip-address:/tmp/

# Using rsync (faster for large files because it can resume interrupted transfers)
rsync -avz --progress flask-docker-app-1.0.0.tar.gz user@your-vm-ip-address:/tmp/

# Using scp with a specific SSH key
scp -i ~/.ssh/my-key.pem flask-docker-app-1.0.0.tar.gz user@your-vm-ip-address:/tmp/
```

#### Step 3: On the VM -- Load and Run

```bash
# SSH into the VM
ssh user@your-vm-ip-address

# Load the image from the tar file
docker load -i /tmp/flask-docker-app-1.0.0.tar.gz

# If you saved without gzip:
docker load -i /tmp/flask-docker-app-1.0.0.tar

# Verify the image is loaded
docker images flask-docker-app

# Run the container
docker run -d \
    -p 80:5000 \
    --name flask-app \
    --restart unless-stopped \
    -e APP_ENV=production \
    flask-docker-app:1.0.0

# Verify
docker ps
curl http://localhost:80/health

# Clean up the tar file to save disk space
rm /tmp/flask-docker-app-1.0.0.tar.gz
```

### Method C: Private Docker Registry

For organizations that want full control over their images without using Docker Hub, running a private registry is the standard approach.

**When to use**: Organizations with multiple developers/servers, CI/CD pipelines, when you need access control over images.

#### Step 1: Set Up a Private Registry on Your VM (or a Dedicated Server)

```bash
# SSH into the server that will host the registry
ssh user@registry-server-ip

# Run the official Docker registry image
docker run -d \
    -p 5000:5000 \
    --name registry \
    --restart unless-stopped \
    -v registry-data:/var/lib/registry \
    registry:2

# Verify the registry is running
curl http://localhost:5000/v2/_catalog
# Expected: {"repositories":[]}
```

#### Step 2: (Optional) Secure the Registry with TLS

For production use, the registry should run behind TLS. Without TLS, Docker will refuse to push/pull unless you explicitly configure it as an insecure registry.

**Option A: Configure as an insecure registry (for testing only)**

On every machine that needs to push/pull from this registry, add it to Docker's insecure registries:

```bash
# Edit or create /etc/docker/daemon.json
sudo tee /etc/docker/daemon.json <<EOF
{
    "insecure-registries": ["registry-server-ip:5000"]
}
EOF

# Restart Docker
sudo systemctl restart docker
```

**Option B: Use a reverse proxy with TLS (for production)**

Put Nginx or Traefik in front of the registry with a valid TLS certificate. This is covered in detail in the Nginx documentation and is beyond the scope of this guide, but the key idea is:

```
Client  -->  Nginx (TLS on port 443)  -->  Registry (port 5000, localhost only)
```

#### Step 3: Push to the Private Registry

```bash
# Tag the image for your private registry
docker tag flask-docker-app:1.0.0 registry-server-ip:5000/flask-docker-app:1.0.0

# Push
docker push registry-server-ip:5000/flask-docker-app:1.0.0

# Verify it is in the registry
curl http://registry-server-ip:5000/v2/_catalog
# Expected: {"repositories":["flask-docker-app"]}

# List tags for a specific image
curl http://registry-server-ip:5000/v2/flask-docker-app/tags/list
# Expected: {"name":"flask-docker-app","tags":["1.0.0"]}
```

#### Step 4: Pull and Run on Any VM

```bash
# On any VM that needs to run the application
ssh user@target-vm-ip

# Pull from the private registry
docker pull registry-server-ip:5000/flask-docker-app:1.0.0

# Run the container
docker run -d \
    -p 80:5000 \
    --name flask-app \
    --restart unless-stopped \
    -e APP_ENV=production \
    registry-server-ip:5000/flask-docker-app:1.0.0

# Verify
curl http://localhost:80/health
```

### Comparison of Methods

| Aspect | Docker Hub | Direct Transfer | Private Registry |
|---|---|---|---|
| **Setup effort** | Minimal (create account) | None | Moderate (run registry server) |
| **Network needed** | Internet access on both sides | SCP/SSH between machines | Network between registry and VMs |
| **Speed** | Depends on internet bandwidth | Fast on local network | Fast on local network |
| **Security** | Public images visible to all; private repos available | Image never leaves your network | Full control, runs on your infrastructure |
| **Scalability** | Excellent (CDN-backed) | Poor (manual for each VM) | Good (any number of VMs can pull) |
| **Best for** | Open source, small teams | Air-gapped, one-off deployments | Organizations, CI/CD pipelines |

---

## 9. Essential Docker Commands Reference

### Container Lifecycle

| Command | Description |
|---|---|
| `docker create <image>` | Create a container without starting it |
| `docker run <image>` | Create and start a container |
| `docker start <container>` | Start a stopped container |
| `docker stop <container>` | Gracefully stop a running container |
| `docker restart <container>` | Restart a container |
| `docker kill <container>` | Forcefully stop a container |
| `docker rm <container>` | Remove a stopped container |
| `docker rm -f <container>` | Force remove (stop + remove) |
| `docker pause <container>` | Pause all processes in a container |
| `docker unpause <container>` | Unpause a paused container |
| `docker rename <old> <new>` | Rename a container |
| `docker wait <container>` | Block until a container stops, then print exit code |

### Container Inspection

| Command | Description |
|---|---|
| `docker ps` | List running containers |
| `docker ps -a` | List all containers (including stopped) |
| `docker ps -q` | List only container IDs |
| `docker logs <container>` | View container logs |
| `docker logs -f <container>` | Follow logs in real time |
| `docker inspect <container>` | View detailed container metadata |
| `docker top <container>` | Show running processes in a container |
| `docker stats` | Live resource usage for all containers |
| `docker stats <container>` | Live resource usage for one container |
| `docker diff <container>` | Show filesystem changes in a container |
| `docker port <container>` | Show port mappings for a container |

### Container Interaction

| Command | Description |
|---|---|
| `docker exec -it <container> /bin/bash` | Open a shell in a running container |
| `docker exec <container> <command>` | Run a command in a running container |
| `docker attach <container>` | Attach to a running container's stdout |
| `docker cp <container>:<path> <host_path>` | Copy files from container to host |
| `docker cp <host_path> <container>:<path>` | Copy files from host to container |

### Image Management

| Command | Description |
|---|---|
| `docker build -t <name>:<tag> .` | Build an image from a Dockerfile |
| `docker images` | List local images |
| `docker pull <image>` | Pull an image from a registry |
| `docker push <image>` | Push an image to a registry |
| `docker tag <src> <dest>` | Create a new tag for an image |
| `docker rmi <image>` | Remove an image |
| `docker history <image>` | Show the layers/history of an image |
| `docker save -o <file>.tar <image>` | Export an image to a tar archive |
| `docker load -i <file>.tar` | Import an image from a tar archive |
| `docker image prune` | Remove dangling images |
| `docker image prune -a` | Remove all unused images |

### Volume Management

| Command | Description |
|---|---|
| `docker volume create <name>` | Create a named volume |
| `docker volume ls` | List all volumes |
| `docker volume inspect <name>` | View volume details |
| `docker volume rm <name>` | Remove a volume |
| `docker volume prune` | Remove all unused volumes |

### Network Management

| Command | Description |
|---|---|
| `docker network create <name>` | Create a network |
| `docker network ls` | List all networks |
| `docker network inspect <name>` | View network details |
| `docker network connect <net> <container>` | Connect a container to a network |
| `docker network disconnect <net> <container>` | Disconnect a container from a network |
| `docker network rm <name>` | Remove a network |
| `docker network prune` | Remove all unused networks |

### System and Cleanup

| Command | Description |
|---|---|
| `docker system df` | Show Docker disk usage |
| `docker system prune` | Remove unused containers, networks, dangling images |
| `docker system prune -a --volumes` | Remove everything unused (nuclear option) |
| `docker info` | Display system-wide Docker information |
| `docker version` | Show Docker client and server versions |

### Docker Compose

| Command | Description |
|---|---|
| `docker compose up` | Create and start services |
| `docker compose up -d` | Start services in detached mode |
| `docker compose up -d --build` | Build images, then start services |
| `docker compose down` | Stop and remove services |
| `docker compose down -v` | Stop, remove services, and delete volumes |
| `docker compose ps` | List running services |
| `docker compose logs` | View logs for all services |
| `docker compose logs -f <service>` | Follow logs for a specific service |
| `docker compose exec <service> <cmd>` | Execute command in a service |
| `docker compose build` | Build images without starting |
| `docker compose pull` | Pull latest images |
| `docker compose restart` | Restart all services |
| `docker compose stop` | Stop services without removing |

---

## 10. Docker Best Practices

### Security

1. **Run as a non-root user**. Always include a `USER` instruction in your Dockerfile. If an attacker exploits a vulnerability in your application and escapes the container, running as non-root limits the damage they can do on the host.

   ```dockerfile
   RUN groupadd --gid 1000 appuser && \
       useradd --uid 1000 --gid 1000 --create-home appuser
   USER appuser
   ```

2. **Use official and verified base images**. Prefer `python:3.12-slim` (official Docker image) over random community images. Official images are maintained, scanned for vulnerabilities, and updated regularly.

3. **Scan images for vulnerabilities**. Use `docker scout` (built into Docker Desktop) or tools like Trivy:

   ```bash
   # Using Docker Scout (Docker Desktop)
   docker scout cves flask-docker-app:1.0.0

   # Using Trivy (open source)
   trivy image flask-docker-app:1.0.0
   ```

4. **Never hardcode secrets in images**. Secrets (API keys, database passwords, tokens) should never appear in a Dockerfile, environment variables baked into the image, or committed to source code. Instead:
   - Pass secrets at runtime via `docker run -e` or `--env-file`
   - Use Docker secrets (in Docker Swarm)
   - Use a secrets manager (HashiCorp Vault, AWS Secrets Manager)

   ```bash
   # Bad: secret baked into the image
   ENV DATABASE_PASSWORD=mysecretpassword

   # Good: secret passed at runtime
   docker run -e DATABASE_PASSWORD="$DB_PASS" my-image

   # Good: secrets from a file (not committed to Git)
   docker run --env-file .env.production my-image
   ```

5. **Keep images up to date**. Rebuild images regularly to pick up security patches in base images and dependencies.

6. **Use read-only filesystems where possible**:

   ```bash
   docker run --read-only --tmpfs /tmp flask-docker-app:1.0.0
   ```

7. **Drop unnecessary Linux capabilities**:

   ```bash
   docker run --cap-drop ALL --cap-add NET_BIND_SERVICE flask-docker-app:1.0.0
   ```

### Image Optimization

1. **Use multi-stage builds** to separate build-time dependencies from runtime dependencies (covered in detail in Section 5).

2. **Order Dockerfile instructions from least to most frequently changing**. This maximizes cache reuse.

3. **Combine RUN commands** to reduce layers:

   ```dockerfile
   # Good: single layer, cleanup in same layer
   RUN apt-get update && \
       apt-get install -y --no-install-recommends curl && \
       rm -rf /var/lib/apt/lists/*
   ```

4. **Use `--no-install-recommends`** with apt-get to avoid pulling in unnecessary packages.

5. **Clean up in the same RUN instruction**. If you install packages and then delete temporary files, both operations must be in the same `RUN` instruction. Otherwise, the temporary files still exist in an earlier layer, contributing to image size.

6. **Use `.dockerignore`** to prevent unnecessary files (Git history, node_modules, virtual environments) from being sent to the build daemon.

### Logging and Monitoring

1. **Write logs to stdout/stderr**, not to files. Docker captures stdout/stderr and makes them available via `docker logs`. If you write to files, you need additional tooling to access them.

   ```python
   # Good: Flask already logs to stderr by default.
   # For gunicorn, use --access-logfile - to log to stdout:
   CMD ["gunicorn", "--access-logfile", "-", "--error-logfile", "-", "app:app"]
   ```

2. **Use `docker stats`** for real-time resource monitoring:

   ```bash
   docker stats
   ```

3. **Use health checks** so you can immediately see if a container is unhealthy:

   ```bash
   docker ps
   # The STATUS column will show "healthy", "unhealthy", or "starting"
   ```

4. **Centralize logs in production** using a logging driver or a log aggregation tool:

   ```bash
   # Use the json-file driver with rotation (default driver, with size limits)
   docker run -d \
       --log-driver json-file \
       --log-opt max-size=10m \
       --log-opt max-file=3 \
       flask-docker-app:1.0.0
   ```

### Development Workflow Best Practices

1. **Use Docker Compose** for local development. It documents your entire stack in a single file and makes `docker compose up` the only command anyone needs to run.

2. **Use bind mounts in development** so code changes are reflected immediately without rebuilding the image.

3. **Use `.env` files** for environment-specific configuration. Keep `.env` out of Git via `.gitignore`.

4. **Tag images deliberately**. Use semantic versioning and/or Git SHAs. Never rely solely on `latest`.

5. **Automate image builds** in CI/CD. The typical flow:
   - Developer pushes code to Git
   - CI pipeline builds the Docker image
   - CI pipeline runs tests inside the container
   - CI pipeline pushes the image to a registry with a version tag
   - CD pipeline deploys the image to staging/production

---

## 11. Practice Exercises

### Exercise 1: Build and Run Your First Container

**Objective**: Get hands-on experience building an image and running a container.

**Steps**:

1. Create a new directory called `exercise-1` and `cd` into it
2. Create a file called `app.py`:
   ```python
   from flask import Flask, jsonify
   app = Flask(__name__)

   @app.route("/")
   def home():
       return jsonify({"message": "Exercise 1 complete!"})

   if __name__ == "__main__":
       app.run(host="0.0.0.0", port=5000)
   ```
3. Create a `requirements.txt`:
   ```
   Flask==3.1.0
   ```
4. Write a `Dockerfile` that:
   - Uses `python:3.12-slim` as the base image
   - Sets the working directory to `/app`
   - Copies and installs requirements
   - Copies the application code
   - Exposes port 5000
   - Runs the app with `python app.py`
5. Build the image: `docker build -t exercise-1:latest .`
6. Run the container: `docker run -d -p 5000:5000 --name ex1 exercise-1:latest`
7. Test it: `curl http://localhost:5000/`
8. View the logs: `docker logs ex1`
9. Stop and remove: `docker rm -f ex1`

**Expected output from curl**: `{"message":"Exercise 1 complete!"}`

---

### Exercise 2: Optimize an Image with Multi-Stage Builds

**Objective**: Understand the size difference between single-stage and multi-stage builds.

**Steps**:

1. In the `exercise-1` directory, copy your Dockerfile to `Dockerfile.single`
2. Create a new `Dockerfile.multi` that uses a multi-stage build:
   - Stage 1 (`builder`): Install dependencies into a venv
   - Stage 2 (`production`): Copy only the venv and app code
3. Build both images:
   ```bash
   docker build -f Dockerfile.single -t exercise-2:single .
   docker build -f Dockerfile.multi -t exercise-2:multi .
   ```
4. Compare their sizes:
   ```bash
   docker images exercise-2
   ```
5. Inspect the layers of each image:
   ```bash
   docker history exercise-2:single
   docker history exercise-2:multi
   ```
6. Write down the size difference and explain why it exists.

**Questions to answer**:
- How much smaller is the multi-stage image?
- What files/tools are present in the single-stage image but absent from the multi-stage image?
- When would the size difference matter most?

---

### Exercise 3: Docker Compose Multi-Service Application

**Objective**: Build a multi-container application with Docker Compose.

**Steps**:

1. Create a new directory called `exercise-3`
2. Create `app.py` -- a Flask app that connects to Redis and counts page views:
   ```python
   import os
   import redis
   from flask import Flask, jsonify

   app = Flask(__name__)
   cache = redis.Redis(
       host=os.environ.get("REDIS_HOST", "redis"),
       port=6379,
       decode_responses=True
   )

   @app.route("/")
   def index():
       count = cache.incr("page_views")
       return jsonify({
           "message": "Hello from Docker Compose!",
           "page_views": count
       })

   @app.route("/health")
   def health():
       try:
           cache.ping()
           return jsonify({"status": "healthy", "redis": "connected"}), 200
       except redis.ConnectionError:
           return jsonify({"status": "unhealthy", "redis": "disconnected"}), 503

   if __name__ == "__main__":
       app.run(host="0.0.0.0", port=5000)
   ```
3. Create `requirements.txt`:
   ```
   Flask==3.1.0
   redis==5.2.1
   ```
4. Write a `Dockerfile` for the Flask app
5. Write a `docker-compose.yml` that defines:
   - A `web` service (your Flask app) that depends on `redis`
   - A `redis` service using `redis:7-alpine`
   - A named volume for Redis data persistence
   - A custom bridge network
6. Start the stack: `docker compose up -d --build`
7. Test it multiple times and observe the page view counter increasing:
   ```bash
   curl http://localhost:5000/
   curl http://localhost:5000/
   curl http://localhost:5000/
   ```
8. Stop the stack, then start it again. Verify the page view counter persisted (because of the volume):
   ```bash
   docker compose down
   docker compose up -d
   curl http://localhost:5000/
   ```
9. Clean up: `docker compose down -v`

---

### Exercise 4: Transfer an Image to a Remote Machine

**Objective**: Practice both methods of getting an image onto a remote machine.

**Steps (you need access to a second machine or VM for this exercise)**:

1. Build the image from Exercise 1:
   ```bash
   docker build -t transfer-test:1.0.0 .
   ```
2. **Method A: Save and load**
   ```bash
   # On your local machine
   docker save transfer-test:1.0.0 | gzip > transfer-test.tar.gz
   ls -lh transfer-test.tar.gz
   scp transfer-test.tar.gz user@remote-host:/tmp/

   # On the remote machine
   ssh user@remote-host
   docker load -i /tmp/transfer-test.tar.gz
   docker run -d -p 5000:5000 --name test transfer-test:1.0.0
   curl http://localhost:5000/
   docker rm -f test
   ```
3. **Method B: Push to Docker Hub and pull**
   ```bash
   # On your local machine
   docker login
   docker tag transfer-test:1.0.0 yourusername/transfer-test:1.0.0
   docker push yourusername/transfer-test:1.0.0

   # On the remote machine
   docker pull yourusername/transfer-test:1.0.0
   docker run -d -p 5000:5000 --name test yourusername/transfer-test:1.0.0
   curl http://localhost:5000/
   docker rm -f test
   ```
4. Compare the two methods: which was faster? Which is more practical for deploying to 10 servers?

**If you do not have a second machine**, you can simulate this by:
- Saving the image to a tar file
- Removing the image from Docker: `docker rmi transfer-test:1.0.0`
- Loading it back from the tar file
- Running it to verify it works

---

### Exercise 5: Security Hardening

**Objective**: Apply security best practices to a Dockerfile.

**Steps**:

1. Start with this intentionally insecure Dockerfile:
   ```dockerfile
   FROM python:3.12
   WORKDIR /app
   COPY . .
   RUN pip install -r requirements.txt
   ENV DATABASE_PASSWORD=supersecret123
   EXPOSE 5000
   CMD ["python", "app.py"]
   ```
2. Identify all the security issues (there are at least 6).
3. Rewrite the Dockerfile fixing every issue. Your fixed version should:
   - Use a minimal base image
   - Use multi-stage builds
   - Create and use a non-root user
   - Not contain any hardcoded secrets
   - Include a `.dockerignore` file
   - Use `--no-cache-dir` for pip
   - Include a `HEALTHCHECK`
   - Pin the base image to a specific version
4. Build both versions and compare:
   ```bash
   docker build -f Dockerfile.insecure -t exercise-5:insecure .
   docker build -f Dockerfile.secure -t exercise-5:secure .
   docker images exercise-5
   ```
5. Scan both images for vulnerabilities (if you have Docker Scout or Trivy available):
   ```bash
   docker scout cves exercise-5:insecure
   docker scout cves exercise-5:secure
   ```
6. Document the security issues you found and how you fixed each one.

**Security issues to find**:
1. Using the full `python:3.12` base image (larger attack surface)
2. Running as root (no `USER` instruction)
3. Hardcoded secret in `ENV DATABASE_PASSWORD`
4. No `.dockerignore` (may copy `.git`, `.env`, etc. into the image)
5. Pip cache not disabled (wasted space, potential information leak)
6. No health check
7. Using Flask dev server instead of gunicorn for production
8. `COPY . .` before `pip install` (poor cache usage, not a security issue but a best practice issue)

---

## Next Steps

After completing this Docker guide and the exercises, you are ready to move on to:

- **Docker Compose in depth** -- Complex multi-service applications, environment management, production compose files
- **Container Orchestration with Kubernetes** -- Managing containers at scale across clusters of machines
- **CI/CD Pipelines** -- Automating Docker builds and deployments with Jenkins, GitHub Actions, or GitLab CI
- **Container Security Scanning** -- Integrating Trivy or Snyk into your pipeline
- **Docker Swarm** -- Docker's built-in orchestrator (simpler alternative to Kubernetes for smaller deployments)
