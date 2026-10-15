# Docker Architecture

## 1. Overview
- Docker uses a **client-server architecture**.
- The **client** sends commands.
- The **daemon** (server) does the work.
- They communicate through a **REST API** over a Unix socket or network.
- **Images** are stored in a **registry**.
- This design allows you to manage Docker locally or remotely.

## 2. Components

### Docker Client (CLI)
- The `docker` command you type in the terminal.
- Sends requests to the Docker daemon via the REST API.
- Can connect to a local or remote daemon.
- Examples: `docker run`, `docker build`, `docker pull`.

### Docker Daemon (dockerd)
- The background service that does the actual work.
- Listens for API requests.
- Manages:
  - Images (build, pull, push, store)
  - Containers (create, start, stop, remove)
  - Networks
  - Volumes
- Uses lower-level tools:
  - **containerd**: manages container lifecycle.
  - **runc**: creates and runs containers (OCI runtime).
- Runs as root by default (can be rootless).

### Docker Engine
- The core of Docker.
- Includes:
  - Docker daemon (`dockerd`)
  - REST API
  - Command-line interface (CLI)
- Often “Docker Engine” refers to the daemon + API + CLI package installed on a host.

### Docker Registry
- A storage and distribution system for Docker images.
- Default public registry: Docker Hub (`hub.docker.com`).
- Can be private (e.g., AWS ECR, Google Artifact Registry, Harbor, GitLab Registry).
- `docker pull` downloads images from a registry.
- `docker push` uploads images to a registry.

## 3. How They Work Together

**Flow for `docker run nginx`:**
1. You type `docker run nginx` in the CLI.
2. CLI sends a request to the Docker daemon via REST API.
3. Daemon checks if the `nginx` image exists locally.
4. If not, daemon pulls it from the configured registry (default: Docker Hub).
5. Daemon uses containerd to create a container from the image.
6. containerd uses runc to start the container process.
7. Daemon sets up networking, volumes, and cgroup limits.
8. Container runs. CLI shows output or returns container ID.

**Diagram (simplified):**
```
[ Docker CLI ]  --REST API-->  [ Docker Daemon ]  --pull/push-->  [ Registry ]
                                      |
                                      v
                              [ containerd ]
                                      |
                                      v
                                 [ runc ]
                                      |
                                      v
                               [ Container ]
```

## 4. Key Commands

```bash
# Check client and daemon versions
docker version

# Check daemon info (storage, runtime, etc.)
docker info

# Pull an image from registry
docker pull nginx:1.25

# Push an image to registry
docker push myrepo/myapp:1.0

# Run a container
docker run -d --name web -p 8080:80 nginx

# List running containers
docker ps

# List images
docker images
```

## 5. Important Notes
- Docker daemon must be running for any Docker command to work.
- On Linux, the daemon runs as a systemd service: `systemctl status docker`.
- The CLI can talk to a remote daemon using `DOCKER_HOST` environment variable.
- Example: `export DOCKER_HOST=ssh://user@remote-host`
- Rootless Docker runs the daemon as a non-root user for better security.
- containerd and runc are part of the container runtime stack, but you rarely interact with them directly.

## 6. Interview Points

### Common Questions & Answers
**Q: What is Docker architecture?**  
A: Client-server architecture. The Docker CLI sends commands via REST API to the Docker daemon, which manages images, containers, networks, and volumes. Images are stored in a registry.

**Q: How does Docker client communicate with the daemon?**  
A: Through a REST API over a Unix socket (`/var/run/docker.sock`) or TCP. The CLI uses this API to send instructions.

**Q: What is the role of the Docker daemon?**  
A: It listens for API requests and performs all container operations: building images, running containers, managing networks and volumes.

**Q: What is a Docker registry?**  
A: A storage system for Docker images. Docker Hub is the default public registry. Private registries include ECR, Harbor, and GitLab.

**Q: What is the difference between Docker Engine and Docker CLI?**  
A: Docker Engine is the core system (daemon + API + CLI). Docker CLI is the command-line client that talks to the daemon.

**Q: What is containerd?**  
A: A container runtime that manages the lifecycle of containers. Docker daemon uses containerd to create, start, and stop containers.

**Q: What is runc?**  
A: A low-level OCI runtime that actually creates and runs containers using Linux namespaces and cgroups.

**Q: What happens when you run `docker run`?**  
A: CLI → REST API → daemon → checks local image → pulls from registry if missing → containerd creates container → runc starts it → networking and limits applied.

### Product-Based Company Scenarios
**Scenario: Docker commands fail with “Cannot connect to the Docker daemon”. What do you check?**  
Check if the daemon is running: `systemctl status docker`. Start it: `systemctl start docker`. Check socket permissions: `/var/run/docker.sock`. Check `DOCKER_HOST` if using remote.

**Scenario: You need to use a private registry for images. How do you configure Docker?**  
Login: `docker login registry.example.com`. Then pull/push with the full image name: `docker pull registry.example.com/myapp:1.0`.

**Scenario: You want to run Docker CLI on your laptop but daemon on a remote server. How?**  
Set `DOCKER_HOST` to the remote daemon: `export DOCKER_HOST=ssh://user@server`. Ensure SSH access and Docker socket permissions.

**Scenario: Image pulls are slow. How do you speed them up?**  
Use a local registry mirror, enable registry caching, use smaller base images, or pull from a closer region.

### Service-Based Company Scenarios
**Scenario: Client asks how Docker works under the hood. What do you explain?**  
Explain the client-server model: CLI sends commands to daemon via REST API. Daemon uses containerd and runc to manage containers. Images come from a registry.

**Scenario: Client wants to know where Docker images are stored on disk.**  
On Linux: `/var/lib/docker/` (overlay2, image, container directories). On Windows: `C:\ProgramData\Docker`. Use `docker info` to see the storage driver and root directory.

**Scenario: Client needs to secure the Docker daemon. What do you recommend?**  
Use TLS for remote API, restrict socket permissions, enable rootless mode, use user namespaces, and avoid exposing the daemon publicly.

**Scenario: Client wants to use Docker in a CI/CD pipeline. What architecture point matters?**  
The CI runner needs access to a Docker daemon. It can be local (Docker-in-Docker) or remote (Docker socket mounted). The CLI sends build and push commands to the daemon.

## 7. Quick Takeaway
- Docker = Client + Daemon + Registry.
- CLI sends commands via REST API.
- Daemon does the work using containerd and runc.
- Registry stores and distributes images.
- Docker Engine = daemon + API + CLI.
- Understand this flow to debug and design Docker-based systems.
