# Docker Scenario-Based Interview Questions

---

## Core Concepts

---

### 1. What happens internally when you execute `docker run nginx`?

When I execute `docker run nginx`, the Docker CLI sends the request to the Docker daemon. The daemon first checks whether the `nginx` image exists in the local image store. If it isn't available, Docker pulls the image from Docker Hub by default, downloading all required image layers.

Once the image is available, Docker creates a new container by adding a writable container layer on top of the image's read-only layers. It then allocates networking and, if no network is specified, connects the container to the default bridge network. If any volumes are specified, they are mounted at this stage.

Finally, Docker starts the container by executing the image's configured `ENTRYPOINT` and `CMD`. For the official Nginx image, this launches the Nginx process. As long as that main process continues running, the container remains in the running state.

---

### 2. What is the difference between `docker run` and `docker start`?

- **`docker run`** — Creates a new container from an image and immediately starts it. If the image is not available locally, Docker first pulls it from the registry. Internally, `docker run` is equivalent to `docker create` followed by `docker start`.

- **`docker start`** — Used only for an existing container that is currently stopped. It simply starts that same container again without creating a new one.

---

### 3. What is the difference between a Docker Image and a Docker Container?

A **Docker image** is an immutable, read-only package that contains the application code, runtime, libraries, dependencies, and configuration required to run an application. Since images are immutable, any change requires building a new image.

A **Docker container** is a running instance of that image. When Docker creates a container, it adds a writable layer on top of the image's read-only layers. Any changes made while the container is running — such as creating files or installing packages — are stored in this writable layer.

- One image can be used to create multiple independent containers.
- Stopping a container does not remove it, so it can be started again without recreating it.
- If the container is removed, Docker needs the image to create a new container.

---

### 4. What is the difference between `docker create`, `docker run`, `docker start`, and `docker exec`?

| Command | When to Use |
|---|---|
| `docker create` | When I want to configure a container first and start it later. |
| `docker run` | For creating and starting a new container in one step. |
| `docker start` | To restart an existing container after it has been stopped. |
| `docker exec` | To inspect logs, troubleshoot, verify files, or run administrative commands inside a running container. |

---

### 5. If I execute `docker exec -it nginx bash` and then type `exit`, will the container stop? Why or why not?

**No.** `docker exec` starts a separate process inside an already running container. Exiting that process only terminates the Bash shell, not the container's main process. The container continues running as long as its main process (PID 1) is still running.

---

### 6. A Docker container suddenly exits immediately after starting. What could be the reasons? How would you troubleshoot it?

My first step would be to run `docker ps -a` to confirm that the container has exited and to check its exit status. Next, I would inspect the container logs using `docker logs <container-id>` because they usually reveal application startup errors.

If the logs are not sufficient, I would use `docker inspect` to review the container configuration, including environment variables, entrypoint, command, mounted volumes, networking, and other runtime settings.

**Common causes:**
- The main application process exiting
- An incorrect `CMD` or `ENTRYPOINT`
- Missing environment variables
- Configuration errors
- Database connection failures
- Permission issues
- Application startup failures

**Common exit codes:**

| Code | Meaning |
|---|---|
| `0` | Program completed successfully |
| `1` | Application error |
| `137` | Killed (often OOM or SIGKILL) |
| `143` | Graceful termination (SIGTERM) |

---

### 7. The container exits immediately and `docker logs` shows nothing useful. What would you do next?

If `docker logs` is empty, I would first use `docker inspect` to verify the runtime configuration — command, entrypoint, environment variables, mounted volumes, and networking. I would also compare the image and configuration with the last known working version to identify any recent changes.

If I still couldn't determine the issue, I would start the image with an overridden entrypoint to keep the container running:

```bash
docker run -it --entrypoint /bin/sh <image>
```

This allows me to inspect:
- Files and filesystem structure
- Environment variables (`env`)
- Permissions (`ls -l`)
- Mounted volumes
- Application binaries
- Configuration files

> This is one of the most common production debugging techniques.

---

### 8. Why does a Docker container stop automatically? What determines whether a container keeps running or exits?

A Docker container remains running **only as long as its main process (PID 1) is running**. If that process exits — whether successfully or due to an error — the container also exits and Docker records the corresponding exit code.

- Commands executed with `docker exec` start additional processes inside an already running container. They do not affect the container's lifecycle.
- The container's lifecycle is controlled entirely by the main process.

**Example:**

```bash
# Exits immediately — ubuntu's default command is not long-running
docker run ubuntu

# Stays alive — bash becomes PID 1 and waits for user input
docker run -it ubuntu bash
```

> `docker exec -it nginx /bin/bash` does **not** keep the container alive. It only starts another process (`bash`) inside an already running container. If the nginx process dies while you're inside Bash, the container will stop.

---

### 9. What is the difference between `CMD` and `ENTRYPOINT`? When would you use each?

| | `CMD` | `ENTRYPOINT` |
|---|---|---|
| **Purpose** | Default command or default arguments | The executable that must always run |
| **Overridable?** | Yes, by providing a command after the image name in `docker run` | Not easily — only via `--entrypoint` flag |
| **When overridden** | Replaced entirely | Command in `docker run` is passed as *arguments* to the ENTRYPOINT |

**Common pattern:** Use `ENTRYPOINT` for the executable and `CMD` for its default arguments.

---

### 10. What is a Docker layer? Why does Docker use layers, and what are the benefits?

A **Docker layer** is a read-only filesystem layer created by most Dockerfile instructions. Docker stacks these layers together to build the final image.

**Benefits:**
- **Build performance** — During a rebuild, Docker reuses cached layers until it encounters a changed instruction. That instruction and all subsequent layers are rebuilt; earlier unchanged layers are reused.
- **Storage efficiency** — Layers are shared between images, so Docker stores common layers only once.
- **Reduced network usage** — When pushing to a registry (Docker Hub, ECR), Docker uploads only the layers not already present.

**Example:**

```
Image A          Image B
---------        ---------
Ubuntu           Ubuntu      ← shared layer (stored once)
Python           Python      ← shared layer (stored once)
App A            App B
```

---

### 11. What is the difference between `docker pull` and `docker run`?

- **`docker pull`** — Downloads an image from a container registry into the local image store. Does **not** create or start any containers.
- **`docker run`** — Creates and starts a new container from an image. If the image is not available locally, Docker automatically pulls it first.

**Note:** When pulling, Docker contacts the registry and checks if the local image matches the remote image for the tag. If already up to date, Docker reuses existing layers and downloads nothing. If newer layers exist, Docker downloads only the missing or changed layers. It does not blindly "update" the image every time.

---

## Practical Docker Usage

---

### 1. The application inside a container is not working properly. The container is still running, but users are getting errors. How would you investigate?

**Investigation order:**

1. `docker ps` — Verify the container is still running.
2. `docker logs <container>` — Check application logs; startup or runtime errors are usually visible here.
3. `docker inspect <container>` — Verify entrypoint, command, environment variables, mounted volumes, networking, and restart policy if logs don't reveal the issue.
4. `docker exec -it <container> /bin/sh` — Enter the container to verify config files, environment variables, file permissions, and manually test the application.
5. `docker top <container>` — Inspect running processes.
6. `docker stats <container>` — Check CPU, memory, and resource usage.

---

#### 1.1 What is the difference between `docker logs` and `docker exec`?

- **`docker logs`** — Views output written by the application's main process to stdout/stderr. My **first step** because it quickly reveals startup failures or runtime errors without entering the container.
- **`docker exec`** — Executes commands inside a running container. I use it when I need deeper investigation: verifying environment variables, checking config files, validating mounted volumes, inspecting permissions, or manually testing the application.

> In practice: use `docker logs` first, only use `docker exec` if logs don't provide enough information.

---

### 2. A developer reports: "I think the wrong environment variables were passed to the container." How would you verify this?

1. `docker ps` — Verify the container is running.
2. `docker inspect <container>` — Review the **`Env`** section and compare with expected values.

While reviewing the inspect output, I would also check:
- Image being used
- Configured `ENTRYPOINT` and `CMD`
- Mounted volumes
- Network configuration
- Port bindings

If an environment variable is missing or incorrect, I would recreate the container with the correct `-e` values:

```bash
docker run -e DB_HOST=mysql -e APP_ENV=prod nginx
```

---

#### 2.1 What is the difference between `docker inspect` and `docker exec`?

- **`docker inspect`** — My **first choice**. Verifies runtime configuration without entering the container: environment variables, mounted volumes, network config, port mappings, image details, `ENTRYPOINT`, and `CMD`.
- **`docker exec`** — Used when the runtime configuration looks correct but the issue persists. Allows inspecting config files, environment from within the application, permissions, connectivity, or manually running application commands.

---

### 3. The container is running but the application is very slow. Sometimes it responds, sometimes it hangs. No errors in `docker logs`.

1. `docker stats` — Monitor CPU, memory, network, and I/O usage to determine if the issue is resource exhaustion.
2. `docker top` — Identify processes inside the container consuming excessive CPU or memory.
3. `docker exec` — If resource usage appears normal, investigate inside the container: application logs, config, active connections, or manually test the application.

---

#### 3.1 `docker stats` shows CPU at 99% but memory only at 180MiB / 2GiB. What next?

High CPU with low memory usage indicates a **CPU-bound** issue, not a memory issue.

**Next steps:**
- Use `docker top` to identify the specific process consuming the CPU.
- Investigate inside the container for infinite loops, excessive request processing, traffic spikes, or inefficient application logic.
- Determine whether this behavior is new or expected for the workload.
- Only increase CPU resources after confirming the application genuinely needs more CPU and the issue is not caused by an application defect or abnormal workload.

---

### 4. What is the difference between `docker stop` and `docker kill`?

| | `docker stop` | `docker kill` |
|---|---|---|
| **Signal sent** | `SIGTERM` (then `SIGKILL` after timeout) | `SIGKILL` immediately |
| **Cleanup** | Application can complete work, close files, release resources | No cleanup — immediate termination |
| **When to use** | Always preferred in production | Only when a container is unresponsive or fails to stop gracefully |

---

#### 4.1 Scenario: A Java application is processing a financial transaction and you run `docker kill payment-service`.

Using `docker kill` during a financial transaction can be dangerous — it immediately terminates the application without allowing it to complete the current operation. This could leave transactions in an **inconsistent state**, interrupt users, or require manual recovery.

`docker stop` is the correct choice. It sends `SIGTERM` first, giving the application an opportunity to complete ongoing requests, close database connections, flush logs, and shut down gracefully.

---

## Dockerfile Best Practices

---

### 5. Write a production-ready Dockerfile for a Spring Boot microservice.

```dockerfile
FROM eclipse-temurin:17-jre-alpine
WORKDIR /app

RUN addgroup -S appgroup && adduser -S appuser -G appgroup

COPY target/app.jar /app/app.jar

RUN chown -R appuser:appgroup /app
USER appuser

EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]
```

**`CMD` vs `ENTRYPOINT` for a Spring Boot JAR:**

- `ENTRYPOINT` defines the main executable that always runs. It cannot be easily overridden.
- `CMD` defines default arguments passed to the `ENTRYPOINT`. It is always appended to `ENTRYPOINT` unless explicitly overridden.

> If you use `CMD ["java", "-jar", "app.jar"]` and run `docker run -it my-image bash`, it drops you into a **Bash shell** — not the Spring Boot app — because `CMD` is entirely replaced by the `bash` argument. With `ENTRYPOINT`, `bash` would be passed as an *argument* to `java`.

---

### 6. What causes dangling images in a CI/CD pipeline? How do you remove them safely?

A **dangling image** is created when you build a Docker image with the same tag. For example, if you build `sbi-app:latest` twice, Docker strips the tag from the old image and assigns it to the new one. The old, untagged image becomes `<none>:<none>` — a dangling image.

**Remove only dangling images:**
```bash
docker image prune
# or explicitly:
docker rmi $(docker images -f "dangling=true" -q)
```

**Why `docker system prune -af` is dangerous in production:**

The `-a` flag removes **ALL unused images**, not just dangling ones. This includes every intermediate build cache and every old version of base images not currently running. In a production environment (e.g., with Amazon ECR), this could remove stable previous tags needed for rapid rollbacks.

> Never run `docker system prune -af` in production. Use `docker image prune` (without `-a`) to safely remove only dangling images.

---

### 7. How do you cache Maven dependencies in Docker so re-downloads only happen when `pom.xml` changes?

The trick is using Maven's `dependency:go-offline` goal, which downloads all dependencies from `pom.xml` into the local Maven cache **without needing the source code**.

```dockerfile
# 1. Copy ONLY the dependency definition file first
COPY pom.xml /app/pom.xml

# 2. Download ALL dependencies without compiling (cached as a layer)
RUN mvn dependency:go-offline

# 3. Copy the entire source code (invalidates cache only when code changes)
COPY src /app/src

# 4. Compile and package (uses cached dependencies from step 2)
RUN mvn clean package
```

**Why this works:**
- Step 2 downloads ~100MB of dependencies into Docker's layer cache.
- When a Java source file changes, Steps 1 and 2 are **cache hits** — only Steps 3 and 4 re-run (~10 seconds instead of 5 minutes).
- When `pom.xml` changes, Step 2 re-runs and re-downloads dependencies.

> If you write `COPY pom.xml` and then run `RUN mvn clean package` without copying `src/`, the build crashes with "Source directory does not exist" because Maven has no code to compile.

---

### 8. You build `sbi-app:latest` locally. Running `docker push sbi-app:latest` fails. Why? What is the fix?

**Reason:** Amazon ECR does not use usernames. It uses the AWS Account ID and Region in the registry domain. The tag `sbi-app:latest` has no ECR registry URL.

**Fix:**

```bash
# 1. Tag with the full ECR registry URL
docker tag sbi-app:latest 123456789012.dkr.ecr.us-east-1.amazonaws.com/sbi-app:latest

# 2. Authenticate with AWS ECR
aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin 123456789012.dkr.ecr.us-east-1.amazonaws.com

# 3. Push
docker push 123456789012.dkr.ecr.us-east-1.amazonaws.com/sbi-app:latest
```

> **Best practice:** Never use `latest` in production. Use the Git commit SHA (e.g., `sbi-app:abc123f`) or the CI build number (e.g., `sbi-app:build-42`) for precise version tracking and instant rollbacks.

---

### 9. You have 5 Spring Boot microservices sharing the same base setup. How do you avoid duplicating 20 lines of Dockerfile?

Create a **custom base image** for the team.

**`Dockerfile.base`:**
```dockerfile
FROM eclipse-temurin:17-jre-alpine
# Set timezone to IST
# Create non-root user
# Install CA certificates
```

Build and push to ECR:
```bash
docker build -f Dockerfile.base -t 123456789012.dkr.ecr.us-east-1.amazonaws.com/sbi-base-java:17 .
docker push 123456789012.dkr.ecr.us-east-1.amazonaws.com/sbi-base-java:17
```

Each microservice Dockerfile then starts with just:
```dockerfile
FROM 123456789012.dkr.ecr.us-east-1.amazonaws.com/sbi-base-java:17
```

**Benefits:**
- Microservice Dockerfiles are 5–6 lines.
- Updating CA certificates or JDK version requires changing only the base image — all 5 microservices inherit the fix on their next build.
- In CI/CD, a separate pipeline builds the base image on schedule or when `Dockerfile.base` changes.

---

### 10. Your image is 400MB. What is wrong with splitting `apt-get` commands into separate `RUN` instructions?

**Problem 1 — Layer bloat:** Each `RUN` creates a separate layer. Files deleted in a later layer still exist in the earlier layer, so the total image size doesn't shrink.

**Problem 2 — Stale cache:** If `apt-get update` is in its own cached layer, running a new build weeks later reuses the outdated package list, which can cause 404 errors or install wrong versions.

**Fix — combine into a single `RUN` command:**

```dockerfile
RUN apt-get update && apt-get install -y --no-install-recommends \
    curl \
    netcat \
    vim \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/*
```

- `--no-install-recommends` — Avoids pulling in unnecessary suggested packages.
- `apt-get clean && rm -rf /var/lib/apt/lists/*` — Deletes downloaded package files **in the same layer** so the image size actually shrinks.

---

### 11. What is the difference between `COPY` and `ADD`? Which should you use for copying a local `requirements.txt`?

| | `COPY` | `ADD` |
|---|---|---|
| **Copies local files** | ✅ | ✅ |
| **Downloads remote URLs** | ❌ | ✅ |
| **Auto-extracts tarballs** | ❌ | ✅ |
| **Recommended for local files** | ✅ | ❌ |

**Always use `COPY` for local files.** The Docker documentation explicitly recommends `COPY` over `ADD` unless you specifically need URL downloading or tarball auto-extraction.

> For a banking client like SBI, using `ADD` with a remote URL means downloading code from the internet at build time — a security risk that bypasses internal artifact repositories.

---

### 12. What is the difference between `ENV` and `ARG` in a Dockerfile?

| | `ARG` | `ENV` |
|---|---|---|
| **Scope** | Build time only | Build time + runtime (persists in container) |
| **Pass value** | `--build-arg VAR=value` | `-e VAR=value` at `docker run` |
| **Visible in running container?** | ❌ | ✅ |

Use `ENV` for variables the running application needs (e.g., `JAVA_OPTS`, `APP_ENV`). Use `ARG` for variables only needed during the build process (e.g., a version number for downloading a tool).

---

### 13. The image is 700MB because it includes the entire JDK, Maven, and source code. Rewrite using multi-stage builds.

**What is a multi-stage build?** A Dockerfile with multiple `FROM` statements. Each stage can have its own base image. Artifacts can be copied from earlier stages into later stages, and earlier stage layers are **discarded** in the final image.

```dockerfile
# Stage 1: Build stage (full JDK + Maven)
FROM maven:3.8-openjdk-17 AS builder
WORKDIR /app

# Copy only pom.xml first for dependency caching
COPY pom.xml .
RUN mvn dependency:go-offline

# Copy source code and build the JAR
COPY src/ ./src/
RUN mvn clean package -DskipTests

# Stage 2: Runtime stage (lightweight JRE only)
FROM eclipse-temurin:17-jre-alpine
WORKDIR /app

# Copy only the JAR from the builder stage
COPY --from=builder /app/target/app.jar /app/app.jar

# Create non-root user for security
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
RUN chown -R appuser:appgroup /app
USER appuser

EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]
```

The final image is **~180MB** — it contains only the JRE and the JAR, not Maven, the JDK, or source code.

---

### 14. Can I use a different base image in each stage? What happens to builder stage layers after the build?

**Yes.** You can use any base image per stage. The only constraint is **compatibility** — build artifacts must be runnable in the runtime stage. For example, a compiled JAR requires a JRE in the final stage; you can't copy it to a `scratch` image.

**Builder stage cache behavior:**
- Builder stage layers are **not** included in the final image but are **cached locally**.
- This means the next build reuses the `mvn dependency:go-offline` layer, drastically speeding up builds.
- To free disk space from cached builder layers: `docker builder prune`

---

### 15. Do I have to name my stages with `AS builder`? What if I don't?

**No, naming is optional.** If you skip `AS builder`, stages are referenced by their **index number** (starting from 0):

```dockerfile
COPY --from=0 /app/target/app.jar /app/app.jar
```

However, **naming is strongly recommended** as best practice:
- Index numbers break if you reorder or add stages.
- Named stages (`AS builder`) are self-documenting and maintainable.

---

## Docker Storage

---

### 16. What is the difference between a Docker volume and a bind mount?

| | **Volume** | **Bind Mount** |
|---|---|---|
| **Managed by** | Docker | You (the host OS) |
| **Stored at** | `/var/lib/docker/volumes/` | Any host path you specify |
| **Portability** | High — no dependency on host path | Low — tied to specific host path |
| **Permissions** | Docker handles it | You manage SELinux, ownership, etc. |

**For SBI microservices:** Use **volumes** because they're portable, easier to back up and migrate, and more secure (host path is abstracted).

**Use bind mounts** for legacy logging agents or tools that must write to a specific, known host path.

---

### 17. Your SBI microservice generates PDF reports saved to `/app/reports/` inside the container. How do you persist them?

```bash
# Create the volume
docker volume create sbi-reports

# Run the container with the volume mounted
docker run -v sbi-reports:/app/reports my-image
```

Reports are now stored in the `sbi-reports` Docker volume and persist even if the container is stopped or removed.

---

### 18. When you run a new container with the same volume, will the `transactions.log` file still exist?

```bash
docker run -v my-volume:/app/data my-image
```

**Bonus:** What if I use a bind mount instead (`-v /host/path:/app/data`) — will the file persist after the container is deleted?

**Yes, absolutely.** The `transactions.log` file will still exist in both cases — whether using a Docker volume (`my-volume`) or a bind mount (`/host/path`).

Volumes and bind mounts are **external storage** that exists outside the container's ephemeral filesystem. When the container is deleted, the container's writable layer is destroyed, but the volume or bind mount remains untouched. The data stays on the host disk.

When a new container is started with the same volume or bind mount, it mounts that same existing storage and the `transactions.log` file is right there. This is exactly how state is maintained for stateful applications like databases or, in SBI's case, report generation and log aggregation services.

---

### 19. The SBI cloud team notices high ECR storage costs due to many unused volumes on EKS worker nodes. How do you clean them up?

`docker volume prune` removes all volumes not referenced by any running container. Add the `-f` flag to skip the confirmation prompt in CI/CD scripts:

```bash
docker volume prune -f
```

> **Caution for SBI's banking environment:** Some volumes might contain critical data even if no container is currently using them (e.g., a database temporarily stopped). Always list volumes with `docker volume ls` and review them before pruning. In production, prefer a lifecycle policy in Terraform or EBS snapshots over manual prune commands.

---

### 20. What does the `VOLUME` instruction in a Dockerfile actually do? What is the downside for sensitive audit logs?

`VOLUME ["/app/logs"]` does **NOT** create a persistent, named volume automatically. Instead, Docker creates an **anonymous volume** with a randomly generated name (like `83f2d...`) and mounts it to `/app/logs`.

**The downside for SBI's banking audit logs:**

Since the volume is anonymous, logs are written to this unnamed volume. When the container is deleted without `docker rm -v`, that anonymous volume becomes a **dangling volume** — still on disk, but with no traceability back to a specific deployment.

**Best practice for SBI:** Never use `VOLUME` in the Dockerfile for application logs. Instead, let the app write to `/app/logs` inside the container filesystem and **explicitly mount a named volume at runtime**:

```bash
docker run -v my-audit-logs:/app/logs my-image
```

This ensures the storage location is predictable and traceable — crucial for banking compliance.

---

## Docker Networking

---

### 21. Docker Networking — The Three Main Network Drivers

| Driver | Description |
|---|---|
| **`bridge`** (default) | Docker creates this automatically. Containers without a specified network attach here. |
| **`host`** | Container uses the host machine's network stack directly — no isolation. Port `8080` in the container is directly exposed on the host's IP without any `-p` mapping. |
| **`overlay`** | Connects containers across multiple Docker hosts (Swarm or Kubernetes). Not needed for single-host setups. |

**The default bridge trap:** Containers on the default bridge network can only communicate by **IP address**, not by container name. To enable DNS-based name resolution between containers, create a **user-defined bridge network**.

**Port mapping rule (`-p 8080:8080`):**
- Left side (`8080`) → host port (your laptop/VM)
- Right side (`8080`) → container port

**The Spring Boot binding trap:** If your app listens on `127.0.0.1` inside the container, it will never accept traffic from the host even with `-p 8080:8080`. Your app **must** bind to `0.0.0.0` (all interfaces). In Spring Boot: `server.address=0.0.0.0` in `application.properties`.

---

### 22. You run a container with `-p 8080:8080` but `http://localhost:8080` fails. Why?

```bash
docker run -d -p 8080:8080 --name my-app my-image
```

The three common Docker network drivers are **bridge**, **host**, and **overlay**. The default is **bridge**.

**Two likely causes of failure:**

1. The Spring Boot app is bound to `127.0.0.1` instead of `0.0.0.0` inside the container. Fix: set `server.address=0.0.0.0` in `application.properties`.

2. On Docker Desktop (Windows/Mac), Docker runs inside a VM. Verify the port mapping is actually registered:

```bash
docker port my-app
```

---

### 23. Connect `sbi-database` and `sbi-backend` on a custom network so they can communicate by hostname.

```bash
# 1. Create the custom bridge network
docker network create sbi-network

# 2. Run the PostgreSQL database
docker run -d --name sbi-database --network sbi-network postgres

# 3. Run the Spring Boot backend on the same network
docker run -d --name sbi-backend --network sbi-network my-spring-app
```

Because this is a **user-defined bridge network**, the backend container automatically resolves the hostname `sbi-database` to the database container's IP — Docker's internal DNS handles this.

---

### 24. What is the difference between `-p` and `EXPOSE`? Which one is actually required to talk to the container from your laptop?

| | `EXPOSE` | `-p` |
|---|---|---|
| **What it does** | Documentation/metadata — tells developers which port the container listens on | Creates the actual port binding between host and container |
| **Opens port to host?** | ❌ No | ✅ Yes |
| **Required for external access?** | ❌ No | ✅ Yes |

- `EXPOSE 8080` does **not** open the port to the host. It also enables the `-P` flag (capital P), which publishes all exposed ports to random host ports.
- `-p 8080:8080` creates the actual port binding. Without it, no traffic from your laptop can reach the container regardless of `EXPOSE`.

> **Analogy:** `EXPOSE` is like labeling a door with a room number. `-p` is like actually cutting a hole in the wall so people can walk through. Without `-p`, the label is useless for external access.

---

### 25. You accidentally created 500 unused networks via CI/CD automation. How do you clean them up?

```bash
docker network prune -f
```

The `-f` flag skips the confirmation prompt — useful for CI/CD scripts.

**What it removes:** All custom user-defined networks not currently used by any running container.  
**What it preserves:** The default `bridge`, `host`, and `none` networks — they are built-in and protected.

> For SBI's production environment, first review unused networks with `docker network ls` before pruning. In CI/CD, schedule this clean-up during off-peak hours.

---

## Docker Compose

---

### 26. What is Docker Compose? Write a `docker-compose.yml` for a Spring Boot backend and PostgreSQL database.

Docker Compose is a tool for defining and running multi-container Docker applications. It allows spinning up an entire stack with a single `docker-compose up` command.

```yaml
version: '3.8'

services:
  database:
    image: postgres:15-alpine
    environment:
      POSTGRES_USER: admin
      POSTGRES_PASSWORD: secret
      POSTGRES_DB: sbi_bank
    ports:
      - "5432:5432"           # Optional: to access DB from host
    volumes:
      - postgres_data:/var/lib/postgresql/data  # Persist data
    networks:
      - sbi-network

  backend:
    build: .                  # Builds from Dockerfile in current directory
    ports:
      - "8080:8080"
    environment:
      SPRING_DATASOURCE_URL: jdbc:postgresql://database:5432/sbi_bank
      SPRING_DATASOURCE_USERNAME: admin
      SPRING_DATASOURCE_PASSWORD: secret
    depends_on:
      - database              # Waits for DB to start (but not fully ready)
    networks:
      - sbi-network

networks:
  sbi-network:
    driver: bridge

volumes:
  postgres_data:
```

**Key points:**

- **Networking** — No need to manually create the network. Compose creates `sbi-network` automatically and attaches both containers.
- **DNS Resolution** — On a user-defined network, Compose provides automatic DNS. The backend reaches the database using the service name `database` as the hostname — no IP addresses needed.
- **Data Persistence** — The `postgres_data` volume persists database data across container restarts.

> `depends_on` only waits for the database **container to start**, not for PostgreSQL to be **fully ready**. For production, add a health check or a wait script in the entrypoint.

---

### 27. How do you rebuild and restart just the backend service without restarting the database?

```bash
docker compose up -d --build backend
```

> `--force-recreate` only recreates the container even if configuration hasn't changed — if you changed Java code and rebuilt the JAR, the Docker image hasn't been rebuilt, so `--force-recreate` restarts the container with the **old JAR**. Use `--build` to rebuild the image first.

---

### 28. How do you override `docker-compose.yml` for production-like testing without modifying the original file?

Docker Compose supports multiple compose files via the `-f` flag:

```bash
docker compose -f docker-compose.yml -f docker-compose.prod.yml up
```

The second file overrides the first. In `docker-compose.prod.yml`, change the build context to use `Dockerfile.prod` and update the volume mounts.

Alternatively, Compose automatically looks for a file named `docker-compose.override.yml` and applies it on top of the base file — no `-f` flag needed. This is ideal for separating local dev from production-specific changes.

---

### 30. How do you securely pass the PostgreSQL password to Compose without hardcoding it in the YAML file?

**Option 1 — `.env` file (standard & recommended for local dev):**

Create a `.env` file in the same directory as `docker-compose.yml`:
```
POSTGRES_PASSWORD=mySup3rSecr3t
```

Reference it in the compose file:
```yaml
environment:
  POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
```

Add `.env` to `.gitignore` so it never gets committed to GitHub.

**Option 2 — Host environment variables:**

```bash
export POSTGRES_PASSWORD=mySup3rSecr3t
```

Compose automatically picks it up.

**Option 3 — Secrets management (production):**

For a banking production environment, use **AWS Secrets Manager** or **Kubernetes Secrets** and inject them at runtime via ECS or EKS. The `.env` file is only acceptable for local Compose testing.

---

## Scenario-Based Troubleshooting

---

### 32. Your Spring Boot container on EKS crashes randomly with no useful logs. The host has 8GB RAM. What is killing the container?

The Linux kernel mechanism is the **OOM Killer (Out-Of-Memory Killer)**. When the host runs out of memory, the kernel kills processes consuming excessive memory to protect the system from crashing.

**Fix without changing application code:**

Set a hard memory limit:
```bash
docker run -m 1g my-image
```

Set a soft memory reservation:
```bash
docker run -m 2g --memory-reservation 1.5g my-image
```

Disable OOM Killer for the container (use with extreme caution):
```bash
docker run --oom-kill-disable -m 2g my-image
```

> Avoid `--oom-kill-disable` in production — it can crash the entire node.

**For Spring Boot specifically:** Set JVM heap limits so the JVM doesn't request more memory than the container allows:
```dockerfile
ENV JAVA_OPTS="-Xmx512m"
```
The JVM heap + container overhead must stay below the container's memory limit.

---

### 33. You SSH into an EC2 instance and run `docker ps`. You get: `Cannot connect to the Docker daemon at unix:///var/run/docker.sock`

**Three common causes:**

**1. Docker daemon is not running:**
```bash
systemctl status docker
systemctl start docker
```

**2. Permission issue — your Linux user is not in the `docker` group:**

The socket `/var/run/docker.sock` is owned by `root:docker`. If your user isn't in the group, you get permission denied.
```bash
sudo usermod -aG docker $USER
# Then log out and back in, or run:
newgrp docker
```

**3. Docker socket file is missing or corrupted:**

This can happen if the daemon crashed unexpectedly. Restarting the daemon recreates the socket file.

> **Quick diagnosis:** Run `sudo docker ps`. If it works with `sudo`, it's a permissions problem, not a daemon issue.

---

### 34. Your Spring Boot container starts and immediately exits with no error logs in `docker logs`. What do you do?

**Debugging checklist:**

1. **Check the entrypoint form** — Shell form (`ENTRYPOINT java -jar app.jar`) can cause the container to exit because the shell intercepts signals incorrectly. Use the exec form:
   ```dockerfile
   ENTRYPOINT ["java", "-jar", "app.jar"]
   ```

2. **Run the container interactively** to see the exact error:
   ```bash
   docker run -it my-image /bin/sh
   ```
   Then manually run `java -jar app.jar` inside the shell.

3. **Check for a missing JAR** — The JAR might not be in the `WORKDIR`. Inspect the container's filesystem to confirm the file path.

4. **Check for a background process** — If the main process detaches and runs in the background, the container exits because there is no foreground process. Spring Boot runs in the foreground by default, but a misconfigured startup script could cause this.

---

### 35. You pull an image from ECR on another EC2 instance and get: `denied: User not authorized to perform sts:AssumeRole`

This error means the AWS credentials on the EC2 instance do not have permission to assume the IAM role required to pull from ECR.

**Step-by-step fix:**

1. Authenticate Docker with AWS ECR:
   ```bash
   aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin 123456789012.dkr.ecr.us-east-1.amazonaws.com
   ```

2. Check the IAM role attached to the EC2 instance. It needs the `AmazonEC2ContainerRegistryReadOnly` policy or a custom policy with:
   - `ecr:GetDownloadUrlForLayer`
   - `ecr:BatchGetImage`
   - `ecr:GetAuthorizationToken`

3. Verify which IAM identity is in use:
   ```bash
   aws sts get-caller-identity
   ```

4. **For EKS:** Use **IAM Roles for Service Accounts (IRSA)** to grant the pod permission to pull the image, rather than relying on the EC2 instance role.

