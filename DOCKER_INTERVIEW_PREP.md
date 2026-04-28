# Docker Interview Prep — SRE & Platform Engineer

---

## TABLE OF CONTENTS

1. [Docker Fundamentals](#1-docker-fundamentals)
2. [Docker Architecture — The Full Stack](#2-docker-architecture--the-full-stack)
3. [Images, Layers & Dockerfiles](#3-images-layers--dockerfiles)
4. [Multi-Stage Builds](#4-multi-stage-builds)
5. [Docker Networking](#5-docker-networking)
6. [Docker Volumes & Storage](#6-docker-volumes--storage)
7. [Resource Management & Container Lifecycle](#7-resource-management--container-lifecycle)
8. [Docker Security](#8-docker-security)
9. [Image Optimization & Slow Build Troubleshooting](#9-image-optimization--slow-build-troubleshooting)
10. [Docker Compose](#10-docker-compose)
11. [Common Interview Q&A](#11-common-interview-qa)
12. [Quick Reference Cheat Sheet](#12-quick-reference-cheat-sheet)

---

## 1. Docker Fundamentals

### What is Docker?

Docker is an open-source platform that automates the deployment, scaling, and management of applications using **containerization**. It packages an application together with everything it needs to run — the code, runtime, libraries, system tools, and settings — into a single portable unit called a **container**.

The key promise Docker makes is: *"works on my machine" becomes "works everywhere."* You build the image once and run it on any host that has Docker installed, whether that is a developer's laptop, a CI server, or a production Kubernetes node.

Virtualization creates isolated virtual machines, each with its own full operating system, by emulating hardware through a hypervisor. Containerization isolates processes on a single shared OS kernel using Linux namespaces and cgroups — no hardware emulation involved

### Container vs Virtual Machine

This is one of the most reliable interview questions at any level. The answer requires you to explain the *mechanism*, not just the surface-level comparison.

**How containers achieve isolation:**
Containers use two Linux kernel features — **namespaces** and **cgroups** — to isolate processes from each other and from the host.

- **Namespaces** give each container its own private view of the system: its own process tree (PID namespace), network stack (network namespace), filesystem root (mount namespace), hostname (UTS namespace), and inter-process communication (IPC namespace). From inside the container, it looks like a completely separate machine, even though it is just a process running on the host.
- **cgroups (Control Groups)** limit how much of the physical resources a container is allowed to consume: CPU, RAM, disk I/O, and network bandwidth. This prevents one container from starving all others.

Each container gets several namespaces. The network namespace isolates the network stack — its own IP, ports, and routing, connected to the host via a veth pair. The PID namespace isolates the process tree — the container sees its own processes only, and its main process appears as PID 1 internally. Together with the mount namespace (its own filesystem view), UTS namespace (its own hostname), and IPC namespace (its own shared memory), these give the container the appearance of being a completely separate machine — even though it is just a set of processes on the host kerne

The critical insight is that containers still run on the **same physical hardware and the same host OS kernel** as every other container. There is no separate OS copy. Namespaces create the illusion of isolation, but behind the scenes all containers are sharing the same kernel, the same CPU cores, the same RAM modules.

**How VMs differ:**
A virtual machine uses a **hypervisor** to emulate hardware. Each VM boots its own full operating system kernel. This provides much stronger isolation — if one VM's kernel crashes it does not affect others — but it costs significantly more in memory and startup time because you are running multiple OS instances.

| Aspect | Container | Virtual Machine |
|---|---|---|
| Isolation mechanism | Namespaces + cgroups (kernel features) | Hypervisor (hardware virtualization) |
| OS | Shares host kernel | Each VM has its own kernel |
| Startup time | Milliseconds | Seconds to minutes |
| Memory overhead | Low (no OS copy) | High (full OS per VM) |
| Portability | Image runs anywhere Docker runs | VM image is hypervisor-specific |
| Security isolation | Weaker (shared kernel) | Stronger (separate kernels) |
| Use case | App packaging, microservices, CI/CD | Full OS isolation, legacy apps, multi-OS |

> **Interview tip:** When asked "are containers fully isolated?", say: containers are *logically* isolated through namespaces, but they share the same physical resources and the same kernel as the host. VM-level isolation requires a hypervisor. Most production environments accept container isolation because the convenience and density benefits outweigh the risk — and Kubernetes adds additional security controls on top.

---

## 2. Docker Architecture — The Full Stack

Docker's architecture is layered. Understanding each layer and what it owns tells you exactly where things can go wrong in production.

### The Four Components

```
Docker Client (CLI)
        │  REST API over Unix socket
        ▼
dockerd (Docker Daemon)
        │  gRPC
        ▼
containerd
        │  calls
        ▼
runc  ──────▶  container process starts
        │
containerd-shim becomes parent of container
```

### 1. Docker Client

The Docker client is simply the CLI tool you type commands into — `docker run`, `docker build`, `docker ps`. It does no work itself. It translates your commands into REST API calls and sends them over a Unix socket (`/var/run/docker.sock`) to the Docker daemon. The client can talk to a remote daemon too, which is the basis for CI runners and remote builds.

### 2. Docker Daemon (`dockerd`)

`dockerd` is a long-running Linux service (started by systemd). It exposes the Docker REST API and is responsible for all **high-level management**: building images, managing networks, managing volumes, authentication to registries, and orchestrating how containers relate to each other (e.g., `--link`, user-defined networks).

Importantly, `dockerd` does **not** create containers itself. It delegates all low-level container lifecycle work to `containerd`. This separation is by design — `dockerd` can be restarted without killing your running containers, because the containers live under `containerd`.

### 3. containerd

`containerd` is the industry-standard container runtime. It manages:
- Pulling and unpacking images from registries
- Managing container filesystem snapshots
- Container lifecycle: create, start, stop, delete
- Interacting with storage and network plugins

`containerd` is where actual container management lives. When `dockerd` wants a container created, it makes a gRPC call to `containerd`. `containerd` then calls `runc` to do the real work at the Linux kernel level.

> `containerd` is also used directly by Kubernetes (via the CRI interface), which is why modern Kubernetes clusters do not need Docker installed at all — just `containerd`.

### 4. runc

`runc` is a **low-level OCI (Open Container Initiative) runtime**. It is not a service — it is a binary that runs, does its job, and exits. When called by `containerd`, `runc`:

1. Sets up Linux **namespaces** (PID, Network, Mount, UTS, IPC)
2. Sets up **cgroups** to enforce resource limits
3. Prepares the container filesystem (mounts)
4. Executes the container's main process (e.g., `nginx`, `node app.js`)
5. **Exits immediately** — it births the container and leaves

`runc` is the literal implementation of the OCI runtime spec. Any OCI-compliant runtime (like `crun` or `kata-containers`) can replace `runc`.

### 5. containerd-shim

Because `runc` exits right after starting the container, something needs to stay alive as the container's parent process. That is the **containerd-shim** — a small per-container process created immediately after `runc` starts the container.

The shim is responsible for:
- **Keeping the container alive** — if `containerd` itself restarts (e.g., upgrade), running containers survive because the shim is still there as parent
- **Forwarding stdout/stderr** back to containerd and Docker
- **Capturing the exit code** when the container stops
- **Reaping zombie processes** inside the container

Think of the shim as a guardian that sits between `containerd` and the container process.

### Step-by-Step Container Start Flow

1. You run `docker run nginx`
2. Docker client sends REST API request to `dockerd`
3. `dockerd` asks `containerd`: "create a container from the nginx image"
4. `containerd` calls `runc` to actually create it
5. `runc` sets up namespaces, cgroups, mounts the filesystem, and starts the `nginx` process
6. `runc` creates `containerd-shim` as parent of the nginx process and **exits**
7. `containerd-shim` becomes the guardian of the running container
8. `containerd` reports back to `dockerd`: container is running
9. `dockerd` reports back to the Docker client

### The Restaurant Analogy

| Component | Role |
|---|---|
| **Docker client** | Customer placing an order |
| **dockerd** | Restaurant owner — takes the order, delegates work |
| **containerd** | Restaurant manager — coordinates kitchen |
| **runc** | Chef — cooks the dish (starts the container) and leaves |
| **containerd-shim** | Waiter — serves the dish, tracks it, reports back |
| **Container process** | The dish itself — the running application |

---

## 3. Images, Layers & Dockerfiles

### What is a Docker Image?

A Docker image is an **immutable, layered package** that contains everything needed to run an application. It is not a running thing — it is a blueprint. When you run an image, Docker creates a container from it.

Images are architecture and OS-specific. An `amd64/linux` image will not run natively on an `arm64` platform without emulation.

### Layered Architecture

Docker images are not one monolithic file. They are a stack of **read-only layers**. Each instruction in a Dockerfile that modifies the filesystem creates a new layer:

```
Layer 4 (COPY ./app /app)        <-- your application code
Layer 3 (RUN pip install -r ...)  <-- dependencies
Layer 2 (WORKDIR /app)
Layer 1 (FROM python:3.11-slim)  <-- base OS + Python runtime
```

Layers are **content-addressed** (stored by SHA256 hash) and shared across images. If ten images all start from `python:3.11-slim`, that base layer is only stored once on disk and pulled once from the registry.

When a container runs, Docker adds a **thin writable layer** on top of the read-only image layers. All writes by the running container (log files, temp files, database files) go into this writable layer. When the container is removed, that writable layer is discarded — which is why containers are ephemeral by default.

### Build Cache

Docker caches each layer. When you rebuild an image, Docker checks: *"has anything changed for this layer?"* If not, it reuses the cached layer and skips re-running that instruction. The cache **invalidates at the first changed layer** and rebuilds everything from that point down.

This has a critical practical implication for Dockerfile design. If you copy your source code before installing dependencies:

```dockerfile
# BAD: source code changes every commit → cache breaks at COPY → pip install re-runs every time
COPY . /app
RUN pip install -r requirements.txt
```

```dockerfile
# GOOD: requirements.txt rarely changes → pip install is cached until deps change
COPY requirements.txt /app/
RUN pip install -r requirements.txt
COPY . /app/        # source code changes here don't affect the pip install layer
```

Order your Dockerfile from **least likely to change at the top, most likely to change at the bottom**.

### Dockerfile Instruction Reference

| Instruction | What It Does | Notes |
|---|---|---|
| `FROM` | Sets the base image | Every Dockerfile must start with this |
| `RUN` | Executes a command **during image build** | Creates a new layer |
| `COPY` | Copies local files into the image | Preferred over `ADD` for local files |
| `ADD` | Like `COPY` but also handles URLs and auto-extracts tarballs | Use only when you need tar extraction |
| `WORKDIR` | Sets the working directory for subsequent instructions | Creates the directory if it doesn't exist |
| `ENV` | Sets environment variables in the image | Visible in running container |
| `EXPOSE` | Documents which port the container listens on | Informational only — does not publish the port |
| `CMD` | Default command when container starts | Can be overridden at `docker run` |
| `ENTRYPOINT` | Main executable for the container | `docker run` args become arguments to ENTRYPOINT |
| `HEALTHCHECK` | Defines how Docker tests if container is healthy | Affects `docker ps` status column |
| `USER` | Sets the user for subsequent instructions and container runtime | Use for non-root execution |

### CMD vs ENTRYPOINT vs RUN — The Classic Interview Question

**RUN** executes at **image build time**. It installs packages, compiles code, sets up the environment. Each `RUN` creates an image layer.

**CMD** is the **default command** that runs when the container starts, but it can be completely replaced by passing a command to `docker run`:

CMD and ENTRYPOINT both run when the container starts, but differently. CMD is the default command — fully replaceable. Pass anything to docker run and CMD is ignored. ENTRYPOINT is the fixed executable — arguments you pass at docker run are appended to it, not replacing it.

The best practice is combining them: ENTRYPOINT as the application executable, CMD as default arguments the operator can override. In our .NET API for example — ENTRYPOINT ["dotnet", "TodoApi.dll"] and CMD ["--urls", "http://+:8080"]. Kubernetes can override the port at deploy time without touching the image.

The common mistake I see is using only CMD for everything, which means any docker run override accidentally replaces the entire application command instead of just the arguments."



```bash
docker run myimage python other_script.py   # replaces CMD entirely
```

**ENTRYPOINT** defines the **fixed executable** — it cannot be overridden by command-line arguments, only appended to:

```bash
docker run myimage --debug    # "--debug" becomes an argument to ENTRYPOINT
```

**Best practice** is to combine them: `ENTRYPOINT` defines the executable, `CMD` provides default arguments that the user can override.

```dockerfile
ENTRYPOINT ["dotnet", "TodoApi.dll"]
CMD ["--port", "8080"]           # default args; user can override: docker run myimage --port 9090
```

### COPY vs ADD

Use `COPY` for everything except when you explicitly need tar auto-extraction. `ADD` has hidden behaviour (URL downloads, tar extraction) that makes Dockerfiles harder to reason about. The Docker documentation itself recommends `COPY` by default.

```dockerfile
COPY ./src /app/src          # explicit, predictable
ADD archive.tar.gz /app/     # use only for tar extraction
```

### HEALTHCHECK

A health check tells Docker (and orchestrators like Kubernetes) whether the container is actually working, not just running. Without it, Docker considers any running container "healthy."

```dockerfile
HEALTHCHECK --interval=30s --timeout=10s --retries=3 \
  CMD curl -f http://localhost:8080/health || exit 1
```

This runs every 30 seconds. If it fails 3 consecutive times, the container is marked `unhealthy`. Kubernetes uses the analogous `livenessProbe` / `readinessProbe` rather than the Docker `HEALTHCHECK`.

---

## 4. Multi-Stage Builds

### The Problem Without Multi-Stage

Before multi-stage builds, teams either shipped bloated images that contained the full build toolchain (.NET SDK, Go compiler, Node, npm, etc.), or they maintained complex shell scripts to build artifacts outside Docker and copy them in. Both approaches were fragile.

A .NET API app might produce a 10 MB self-contained binary, but the image containing the full .NET SDK and all build dependencies could easily be 700 MB — all of which ships to production and represents a massive attack surface.

### How Multi-Stage Builds Work

Multi-stage builds let you use multiple `FROM` statements in one Dockerfile. Each `FROM` starts a new stage with a fresh filesystem. You can **selectively copy** artifacts from earlier stages into later stages. Only the final stage becomes the runtime image.

```dockerfile
# ─────── Stage 1: Build ───────
FROM mcr.microsoft.com/dotnet/sdk:9.0 AS build
WORKDIR /src

# Restore dependencies first — cached until .csproj changes
COPY TodoApi.csproj .
RUN dotnet restore

# Copy source and publish a release build
COPY . .
RUN dotnet publish -c Release -o /app/publish --no-restore

# ─────── Stage 2: Runtime ───────
FROM mcr.microsoft.com/dotnet/aspnet:9.0 AS runtime
WORKDIR /app

# Copy only the published output from the build stage
COPY --from=build /app/publish .

# Run as non-root
RUN addgroup --system appgroup && adduser --system --ingroup appgroup appuser
USER appuser

EXPOSE 8080
ENTRYPOINT ["dotnet", "TodoApi.dll"]
```

The final image uses `mcr.microsoft.com/dotnet/aspnet` — the ASP.NET runtime — which is roughly 220 MB, compared to over 700 MB for the full SDK image. The .NET SDK, build caches, and all source code are left behind in the build stage and never reach production. The runtime image is lean, fast to pull, and has a much smaller attack surface.

> **Note on layer caching:** Copying the `.csproj` and running `dotnet restore` *before* copying the full source is the .NET equivalent of the `requirements.txt` trick. NuGet package restore is cached until the project file changes — source code changes do not invalidate the restore layer.

### Why SREs Care

- Smaller images mean **faster pull times** in Kubernetes when nodes auto-scale or containers are rescheduled
- Build tools (compilers, test frameworks) are **never present in production** — a compromised container has fewer tools available to an attacker
- One Dockerfile describes the entire pipeline from source to artifact, which simplifies CI/CD configuration

---

## 5. Docker Networking

### How Docker Networking Works Under the Hood

When Docker creates a container, it creates a new **network namespace** for it — the container gets its own isolated network stack, complete with its own virtual ethernet interface, IP address, routing table, and DNS. From inside the container, the network looks completely private.

To connect that isolated namespace to the outside world, Docker uses a **veth pair** — a virtual patch cable with two ends. One end sits inside the container's namespace (visible as `eth0` inside the container), and the other end connects to a virtual bridge on the host (by default `docker0`).

Port publishing (`-p 8080:80`) is implemented through **iptables NAT rules**. Docker writes rules that forward traffic arriving at port 8080 on the host to port 80 on the container's IP. This is automatic and invisible but has performance implications at high scale.

### Network Drivers

**bridge (default)**
The default mode for standalone containers. Each container gets a private IP on the virtual `docker0` bridge. Containers on the same bridge can communicate by IP. To reach a container from outside the host, you must publish ports. Containers on the *default* bridge network cannot resolve each other by name — only by IP.

**User-defined bridge (recommended over default)**
When you create your own bridge network, Docker automatically provides **DNS-based service discovery**. Containers on the same user-defined network can reach each other using the container name. If a container restarts and gets a new IP, the DNS record updates automatically. This is why you should always use custom networks for multi-container apps.

```bash
docker network create my-app-network
docker run --network my-app-network --name database postgres:15
docker run --network my-app-network --name backend myapp:latest
# backend can now reach database with: psql -h database
```

**host**
The container shares the host's network namespace entirely. There is no network isolation — the container listens directly on the host's ports with no NAT overhead. Useful for performance-sensitive applications but sacrifices network isolation entirely.

**none**
The container has a network stack but is connected to nothing. Completely network-isolated. Used for batch jobs or security-sensitive workloads that require zero network access.

**overlay**
Used for multi-host networking. Required for Docker Swarm services and allows containers running on different physical hosts to communicate as if they were on the same local network. Creates an encrypted, distributed network spanning the cluster. In Kubernetes, the CNI plugin (Calico, Flannel, Cilium) serves the same purpose.

### Default Bridge vs User-Defined Bridge

| Feature | Default Bridge (`docker0`) | User-Defined Bridge |
|---|---|---|
| Container DNS resolution | ❌ IP only | ✅ Resolve by container name |
| Container isolation | Containers share the default bridge | Scoped to your network |
| Connect/disconnect at runtime | ❌ | ✅ `docker network connect` |
| Recommended for production | ❌ | ✅ |

### Networking Troubleshooting Scenarios

**Scenario 1: Container A can't reach Container B by name, but IP ping works.**
This is the classic symptom of being on the **default bridge network**. The default bridge has no DNS. Move both containers to a user-defined network and name resolution will work immediately.

**Scenario 2: `docker run -p 80:80 nginx` fails with "port already allocated."**
Some other process (another container, Nginx on the host, an Apache server) is already using port 80. Check with `sudo lsof -i :80` or `docker ps` to find the conflict. Either stop the conflicting process or remap: `-p 8080:80`.
```bash
# Most useful — shows process name, PID, and user
sudo lsof -i :80

# Modern alternative to netstat
sudo ss -tlnp | grep :80

# Classic netstat
sudo netstat -tlnp | grep :80

# Check if another container has the port
docker ps --format "table {{.Names}}\t{{.Ports}}"

# Minimal — just the PID
sudo fuser 80/tcp
```bash

**Scenario 3: Container intermittently fails to resolve another container's name.**
The container likely restarted and lost its place in the network. Check with `docker network inspect <network>` — look for whether the container is still listed as a network member. Containers that crash and restart reconnect, but if the restart policy is off the container may not have rejoined. Also check if there is a DNS caching issue in the app layer.

**Scenario 4: Microservices on separate bridge networks cannot communicate.**
Each custom bridge network is isolated. Connect a container to multiple networks:
```bash
docker network connect frontend app-container
docker network connect backend app-container
```
Now `app-container` has interfaces on both networks and can bridge communication.

**Scenario 5: High-traffic app experiences network latency inside Docker.**
NAT overhead from port publishing becomes measurable at high request rates. Consider switching to `--network host` if the workload can tolerate reduced network isolation. Also check for MTU mismatch: `docker network inspect <network>` to view the configured MTU, and compare with the host's physical interface MTU — a mismatch causes packet fragmentation.

---

## 6. Docker Volumes & Storage

### The Writable Layer Problem

Every running container has a writable layer on top of the read-only image layers. This layer is managed by the storage driver (typically `overlay2`) and lives at `/var/lib/docker/...` on the host. The problem is that when the container is removed, this writable layer is destroyed with it. Any data written there — database files, uploaded files, log files — is gone.

For anything you need to survive container restarts or replacements, you need persistent storage. Docker provides three mechanisms.

### Named Volumes (Preferred)

Named volumes are managed entirely by Docker. Docker decides where to store them on the host (`/var/lib/docker/volumes/`). You reference them by name, not by host path.

```bash
docker volume create my-db-data
docker run -v my-db-data:/var/lib/postgresql/data postgres:15
```

Named volumes are the **recommended approach** for production data. They are portable, easy to back up, and not tied to a specific host directory structure. They survive container removal and can be shared between containers.

```bash
# Inspect volume location
docker volume inspect my-db-data

# Backup a volume
docker run --rm -v my-db-data:/data -v $(pwd):/backup alpine \
  tar czf /backup/my-db-data.tar.gz /data

# List and clean up unused volumes
docker volume ls
docker volume prune
```

### Bind Mounts

Bind mounts map a **specific path on the host** into the container. You control exactly where the data lives on the host filesystem.

```bash
docker run -v /home/user/app/config:/app/config:ro myapp
```

Bind mounts are used in **development** to mount source code into a container so changes are immediately visible without rebuilding the image. They are also used to inject configuration files from the host.

The downside is that they are tightly coupled to the host's directory structure. Moving the container to a different host requires the same path to exist. They also expose host filesystem paths, which is a security concern in shared environments.

### tmpfs Mounts

A `tmpfs` mount lives only in the host's memory — it is never written to disk. Data inside it disappears when the container stops.

```bash
docker run --tmpfs /app/tmp:size=64m myapp
```

Used for temporary scratch space, secrets that must never touch disk, or performance-critical ephemeral data where disk I/O is the bottleneck.

### Comparison Table

| | Named Volume | Bind Mount | tmpfs |
|---|---|---|---|
| Storage location | Docker-managed (`/var/lib/docker/volumes`) | Any host path you specify | Host RAM only |
| Survives container removal | ✅ | ✅ (host file persists) | ❌ |
| Portable across hosts | ✅ | ❌ (path-dependent) | N/A |
| Best for | Production data, databases | Dev source code, config injection | Secrets, temp files |
| Performance | Good | Good | Excellent (RAM) |
| Host path exposure | ❌ | ✅ (security concern) | ❌ |

### Docker Compose Volumes

```yaml
services:
  db:
    image: postgres:15
    volumes:
      - db-data:/var/lib/postgresql/data     # named volume
      - ./init.sql:/docker-entrypoint-initdb.d/init.sql:ro   # bind mount (read-only)

  app:
    image: myapp:latest
    volumes:
      - app-logs:/app/logs                   # named volume for logs
    tmpfs:
      - /app/tmp                             # tmpfs for scratch

volumes:
  db-data:       # Docker manages this
  app-logs:
```

---

## 7. Resource Management & Container Lifecycle

### CPU and Memory Limits

By default, a container can consume as much CPU and RAM as the host has available. In a multi-container environment this is dangerous — one misbehaving container can starve all others. Set limits explicitly:

```bash
# Memory: hard limit 512 MB, soft limit 256 MB
docker run --memory=512m --memory-reservation=256m myapp

# CPU: limit to 1.5 CPU cores, pin to CPU 0 and 1
docker run --cpus=1.5 --cpuset-cpus=0,1 myapp
```

These limits are enforced by **cgroups** in the Linux kernel. Docker writes the limit values into cgroup configuration files, and the kernel enforces them at the scheduler and memory allocator level.

### OOM Kill — What Happens When Memory Is Exhausted

When a container exceeds its memory limit, the Linux kernel's OOM (Out of Memory) killer terminates the container process. This is not Docker's doing — it is a fundamental kernel protection mechanism.

Symptoms:
- Container exits with **exit code 137** (128 + signal 9, SIGKILL)
- `docker inspect <container>` shows `"OOMKilled": true`
- Docker daemon logs contain OOM entries

```bash
# Check if a container was OOM killed
docker inspect <container> --format='{{.State.OOMKilled}}'

# Check host kernel logs for OOM events
journalctl -k | grep -i "oom\|killed"
```

The container does not necessarily log the OOM event itself — the process is killed externally without warning, so application logs often show nothing. Always check `docker inspect` and host kernel logs when debugging mysterious container exits with code 137.

### Container Lifecycle

Understanding the full lifecycle matters for incidents, automation, and cleanup scripts.

```
docker create  →  [created]
docker start   →  [running]
docker pause   →  [paused]    (SIGSTOP — process frozen, still in memory)
docker unpause →  [running]
docker stop    →  [exited]    (SIGTERM → 10s grace → SIGKILL)
docker kill    →  [exited]    (SIGKILL immediately)
docker rm      →  (removed)   (container and writable layer deleted)
```

### `docker stop` vs `docker kill`

`docker stop` sends **SIGTERM** first. This gives the application time to close connections, flush write buffers, and shut down gracefully. After a configurable grace period (default 10 seconds), if the process has not exited, Docker sends **SIGKILL** to force it.

`docker kill` sends **SIGKILL** immediately, with no warning and no grace period. Data that was in flight but not flushed to disk may be lost.

For production databases, message consumers, or any stateful service, always use `docker stop`. Only reach for `docker kill` when a container is frozen and unresponsive.

```bash
docker stop --time=30 my-db   # give it 30 seconds to shut down gracefully
```

### Restart Policies

| Policy | Behaviour |
|---|---|
| `no` (default) | Never restart automatically |
| `on-failure[:max]` | Restart only if exit code is non-zero. Optional max retry count. |
| `always` | Always restart, including on Docker daemon restart |
| `unless-stopped` | Like `always` but does not restart if manually stopped |

```bash
docker run --restart=on-failure:5 myapp   # up to 5 retries on non-zero exit
```

> **Interview tip:** `unless-stopped` is the most useful policy for long-running services — it survives host reboots but respects intentional `docker stop` commands. `always` is too aggressive for production because it will restart even after deliberate stops.

---

## 8. Docker Security

Security in Docker is not a single feature — it is a set of decisions you make at every layer: the base image, the Dockerfile, the runtime configuration, and the CI/CD pipeline.

### Run as Non-Root

By default, processes inside a container run as **root (UID 0)**. Even though root inside a container is partially constrained by namespaces, it is still a significant risk. A container escape vulnerability (breakout through runc, kernel exploit) gives an attacker full root on the host.

```dockerfile
# Create a dedicated user and switch to it
RUN addgroup --system appgroup && adduser --system --ingroup appgroup appuser
USER appuser
```

In Kubernetes, enforce this with a `securityContext`:
```yaml
securityContext:
  runAsNonRoot: true
  runAsUser: 1000
```

### Drop Linux Capabilities

Linux capabilities split root privileges into granular permissions. Docker grants containers a default set of capabilities, many of which your application almost certainly does not need. Drop all and add back only what is required:

```bash
docker run --cap-drop ALL --cap-add NET_BIND_SERVICE mywebserver
```

`NET_BIND_SERVICE` allows binding to ports below 1024. This is often the only capability a web server needs.

### Use Read-Only Filesystem

If your application does not need to write to its filesystem at runtime (stateless services), mount it read-only. This prevents malware or a compromised process from modifying container contents:

```bash
docker run --read-only --tmpfs /tmp myapp
```

### Image Scanning — Trivy and Docker Scout

Image scanning analyses your image layers for **known CVEs (Common Vulnerabilities and Exposures)** in OS packages, language runtimes, and application dependencies. It does not protect you from zero-days, but it catches the large population of well-documented, patchable vulnerabilities.

**Trivy** (by Aqua Security) is the most widely adopted open-source scanner in SRE toolchains. It scans container images, filesystems, Git repositories, Kubernetes manifests, and IaC configurations:

```bash
# Scan an image
trivy image nginx:latest

# Fail the pipeline on CRITICAL or HIGH vulnerabilities
trivy image --exit-code 1 --severity CRITICAL,HIGH myapp:$BUILD_TAG

# Scan a tarball (e.g., image built in CI before pushing)
trivy image --input myapp.tar
```

**Docker Scout** is Docker's native scanning tool, integrated into Docker Hub and Docker Desktop. It provides CVE scanning, base image recommendations, and a software bill of materials (SBOM):

```bash
docker scout cves myapp:latest
docker scout recommendations myapp:latest   # suggests a better base image
```

**Where scanning fits in CI/CD:**
The right place for image scanning is after the `docker build` step and before the `docker push` step. If vulnerabilities are found above your severity threshold, the pipeline fails and the image is never pushed to the registry. This ensures that only scanned, approved images enter your container registry.

```
docker build → docker scout / trivy scan → (pass) → docker push → deploy
                                          → (fail) → pipeline blocked
```

### Secrets Management — The Right Way

**Anti-patterns: what NOT to do**

Do not put secrets in environment variables if you can avoid it. `ENV SECRET=mysecretvalue` bakes the secret into an image layer permanently — it is visible in `docker history` and in any registry that stores the image. Even if you later override the env var at runtime, the secret remains embedded in the layer.

Do not pass secrets as build arguments (`ARG`). They too appear in `docker history` and in the build cache.

```dockerfile
# NEVER do this
ENV DATABASE_PASSWORD=supersecret123
ARG API_KEY=myapikey
```

**Best practice: External Secrets**

In production, secrets should live in a **dedicated secrets manager**, not in images, environment variable files, or source code. The application retrieves them at startup via the secrets manager SDK or a sidecar.

- **Azure Key Vault** — Secrets are fetched by the application using managed identity or a service principal. In Azure Container Apps or AKS, the [Azure Key Vault provider for Secrets Store CSI Driver](https://learn.microsoft.com/en-us/azure/aks/csi-secrets-store-driver) can mount Key Vault secrets directly as files or env vars into the pod.
- **AWS Secrets Manager** — Applications use the AWS SDK to fetch secrets at runtime. AWS ECS/EKS can inject secrets from Secrets Manager as env vars via task definitions.
- **HashiCorp Vault** — A cloud-agnostic secrets engine. Vault agent sidecar or Vault injector (in Kubernetes) fetches secrets and writes them to an in-memory filesystem accessible by the application.

The pattern is always: **the secret never touches the image**, the running environment retrieves it at startup from an external, audited, access-controlled store.

For local development and Docker Compose, use a `.env` file (not committed to source control) and reference it in `docker-compose.yml`:

```yaml
env_file:
  - .env   # .env is in .gitignore
```

Never commit `.env` files. Use `.env.example` with placeholder values as documentation.

### .dockerignore

Always maintain a `.dockerignore` file. It excludes files from the **build context** — the set of files sent to the Docker daemon when building. Without it, Docker sends your entire project directory including:

- `.git/` — entire git history, potentially containing old secrets
- `node_modules/` or `.venv/` — large dependency directories that slow build context transfer
- `*.log`, `tmp/`, `dist/` — unnecessary files

```
.git
.gitignore
node_modules
*.log
.env
.DS_Store
README.md
```

A smaller build context means faster builds and prevents sensitive files from accidentally landing in the image.

---

## 9. Image Optimization & Slow Build Troubleshooting

### Optimization Checklist

1. **Use minimal base images** — Replace `ubuntu:22.04` or `python:3.11` with `python:3.11-slim` or `python:3.11-alpine`. Alpine-based images can be 5–10x smaller, though Alpine uses `musl libc` which occasionally causes compatibility issues with native extensions.

2. **Combine RUN commands** — Each `RUN` creates a layer. Chaining commands with `&&` reduces the layer count and, more importantly, ensures that installation caches are cleaned in the same layer they were created:

   ```dockerfile
   # Bad: cache cleanup in a different layer has no effect
   RUN apt-get update && apt-get install -y curl
   RUN rm -rf /var/lib/apt/lists/*

   # Good: install and clean up in one layer
   RUN apt-get update && apt-get install -y curl \
       && rm -rf /var/lib/apt/lists/*
   ```

3. **Order layers by change frequency** — Most-stable instructions at the top (base image, dependency installation), most-volatile at the bottom (source code copy). This maximises cache reuse across builds.

4. **Use multi-stage builds** — Ship only the runtime artifact, not the build toolchain. (See Section 4.)

5. **Use `.dockerignore`** — Exclude everything not needed in the image. (See Section 8.)

6. **Do not run as root** — Not strictly an optimization, but an image hygiene practice. (See Section 8.)

### How I Troubleshoot Slow Docker Builds

When I encounter a slow Docker build, I follow a systematic approach.

The first thing I check is **how the Dockerfile uses caching**. I look at whether frequently changing files (like source code) are being copied before stable dependencies (like `package.json` or `requirements.txt`). If source code changes invalidate the dependency installation layer on every commit, every build does a full `npm install` or `pip install`. Moving the `COPY requirements.txt` and `RUN pip install` before `COPY . .` often cuts build time by 60–80% for typical applications.

The second thing I check is the **build context size**. I run `du -sh .` in the project root to see how large the directory is, then verify that `.dockerignore` is excluding `node_modules`, `.git`, build output directories, and test fixtures. Docker transfers the entire build context to the daemon before it starts the build, so a 500 MB context always adds 5–10 seconds even if nothing in it changes.

Next I look at the **base image**. If the team is using `node:18` instead of `node:18-slim`, there may be 200–300 MB of unnecessary OS packages. I switch to a slim or Alpine variant and verify the application still works — occasionally Alpine's `musl libc` breaks native Node.js modules, in which case I use `debian:slim` as a compromise.

For deeper investigation I use `docker history myimage:tag` to see the size of each layer and identify which `RUN` instructions are largest. I also run the build with `--progress=plain` to see the time spent on each step and identify which steps are slow even with a warm cache.

Finally, I make sure **Docker BuildKit is enabled**. BuildKit can build independent stages in parallel and has a more efficient cache subsystem. On modern Docker versions it is enabled by default, but on older setups: `DOCKER_BUILDKIT=1 docker build .` or set `"features": {"buildkit": true}` in `/etc/docker/daemon.json`.

---

## 10. Docker Compose

### What is Docker Compose?

Docker Compose is a tool for defining and running **multi-container applications** using a single YAML file (`docker-compose.yml`). Instead of writing multiple `docker run` commands with long flag lists, you declare your services, networks, and volumes in one file and bring everything up with `docker compose up`.

Compose is primarily used for:
- **Local development environments** — define a full stack (app + database + cache + message queue) locally
- **Integration testing in CI** — spin up dependencies for integration tests, run tests, tear everything down with `docker compose down`
- **Simple staging environments** — not a replacement for Kubernetes but suitable for smaller workloads

### Key Compose Commands

```bash
docker compose up -d            # start all services in background
docker compose down             # stop and remove containers + default networks
docker compose down -v          # also remove named volumes (careful in production!)
docker compose logs -f app      # follow logs for the app service
docker compose ps               # list services and their status
docker compose exec app bash    # shell into running container
docker compose build            # rebuild images
docker compose pull             # pull latest images
docker compose restart app      # restart one service
docker compose scale app=3      # run 3 replicas of app (for stateless services)
```

### Example: 2-Tier Application (Python + PostgreSQL)

```yaml
# docker-compose.yml
version: "3.9"

services:
  app:
    build: .                          # build from Dockerfile in current directory
    image: my-python-api:latest
    ports:
      - "8000:8000"
    environment:
      - DB_HOST=db                    # use service name — Compose DNS resolves it
      - DB_PORT=5432
      - DB_NAME=${POSTGRES_DB}        # reference from .env file
    env_file:
      - .env                          # secrets loaded from .env (never commit this)
    depends_on:
      db:
        condition: service_healthy    # wait for db health check to pass
    networks:
      - backend
    restart: unless-stopped

  db:
    image: postgres:15-alpine
    environment:
      POSTGRES_DB: ${POSTGRES_DB}
      POSTGRES_USER: ${POSTGRES_USER}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}    # read from .env
    volumes:
      - db-data:/var/lib/postgresql/data         # named volume — persists across restarts
      - ./init.sql:/docker-entrypoint-initdb.d/init.sql:ro  # seed script
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER}"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - backend
    restart: unless-stopped

volumes:
  db-data:        # Docker manages storage location

networks:
  backend:
    driver: bridge   # user-defined bridge — DNS resolution by service name works
```

```bash
# .env (NOT committed to source control — add to .gitignore)
POSTGRES_DB=appdb
POSTGRES_USER=appuser
POSTGRES_PASSWORD=changeme
```

Key patterns in this example:
- Services communicate using **service names** (`db`, `app`) — Compose handles DNS on the user-defined network
- `depends_on` with `condition: service_healthy` prevents the app from starting before the database is accepting connections
- Secrets come from `.env`, not hardcoded in the YAML
- A named volume ensures database data survives `docker compose down` (without `-v`)

---

## 11. Common Interview Q&A

---

**"Walk me through the Docker architecture. What happens when you run `docker run nginx`?"**

When you type `docker run nginx`, the Docker CLI translates that into a REST API call and sends it to the Docker daemon (`dockerd`) over a Unix socket. The daemon checks whether the `nginx` image is available locally; if not, it pulls it from Docker Hub via the registry API.

Once the image is available, `dockerd` asks `containerd` to create a container from that image. `containerd` calls `runc`, which does the actual Linux work: setting up namespaces (network, PID, mount, UTS), applying cgroup limits, mounting the container's filesystem layers, and executing the nginx process. `runc` then exits — its job is done.

At this point, `containerd` creates a per-container process called `containerd-shim`, which becomes the parent of the nginx process. The shim keeps the container alive even if `containerd` itself is restarted, and it captures the container's stdout, stderr, and exit code. The container is now running and reporting back to `dockerd` through the shim.

---

**"Why can't Container A reach Container B by name? IP ping works fine."**

This is a classic symptom of using the **default bridge network** (`docker0`). The default bridge network does not include Docker's internal DNS resolver, so containers on it can only reach each other by IP address — not by container name. If a container restarts and gets a new IP, the connection breaks entirely.

The fix is to move both containers onto a **user-defined bridge network**. On a custom network, Docker runs an embedded DNS server that resolves container names to their current IP addresses. Container restarts no longer break communication because the DNS record updates automatically.

```bash
docker network create my-app-net
docker run --network my-app-net --name backend myapp
docker run --network my-app-net --name db postgres
# backend can now connect to: host=db
```

---

**"A container keeps exiting with code 137. What do you do?"**

Exit code 137 means the container process received SIGKILL. The two most common causes are an **OOM kill** (the container exceeded its memory limit) or someone manually running `docker kill`.

I start by checking:
```bash
docker inspect <container> --format='{{.State.OOMKilled}}'
```
If that returns `true`, the container was killed by the kernel OOM killer because it exceeded its memory cgroup limit. I then check `docker stats` to see current memory usage and look at `journalctl -k | grep -i oom` on the host for the kernel's OOM event log.

The resolution depends on the root cause: if the application has a genuine memory leak, that is a code issue. If the limit is legitimately too low for the workload, I increase it (`--memory=1g`). If it is a sudden spike, I might add memory autoscaling in Kubernetes or investigate whether the application buffers data in memory unnecessarily.

---

**"How do you make sure secrets never end up in a Docker image?"**

This is a multi-layer problem. The most common mistake is setting secrets as `ENV` or `ARG` in a Dockerfile — both methods bake the secret into a layer that is visible in `docker history` and in any copy of the image in a registry.

The approach I follow:
1. **Never put secrets in Dockerfiles or committed `.env` files.** The image should contain only the application code and its dependencies.
2. **At runtime**, secrets are injected from an external secrets manager. In Azure environments I use Azure Key Vault — either the application uses the Key Vault SDK with managed identity to fetch secrets at startup, or in Kubernetes I use the CSI Secrets Store Driver to mount Key Vault secrets as files into the pod. In AWS environments, the equivalent is Secrets Manager. In multi-cloud or on-premises environments, HashiCorp Vault is the standard.
3. **For local development**, I use a `.env` file that is in `.gitignore`. I provide a `.env.example` with placeholder values checked into source control so developers know what variables to set.
4. **In CI/CD pipelines**, secrets are stored as pipeline variables (GitHub Actions secrets, Azure DevOps variable groups with Key Vault integration) and injected as environment variables only for the duration of the pipeline run.

The key principle is: the secret never touches disk as part of the image, and the image registry never becomes an accidental secrets store.

---

**"How do you approach image security scanning in a CI/CD pipeline?"**

We integrate image scanning as a **mandatory gate** between the build step and the push step. The pipeline builds the image, scans it, and only pushes to the registry if the scan passes. This ensures no unscanned or non-compliant image ever reaches the registry.

We use **Trivy** for this. It is fast, integrates well with CI runners, and covers OS packages, language runtimes, and application dependencies. The pipeline step looks like:

```bash
trivy image --exit-code 1 --severity CRITICAL,HIGH myapp:$BUILD_TAG
```

The `--exit-code 1` causes the pipeline to fail if any CRITICAL or HIGH CVE is found. We address CRITICAL vulnerabilities before merging. HIGH vulnerabilities get a tracked ticket with a resolution deadline.

For base image freshness, **Docker Scout** gives useful recommendations — it tells you if a newer patch version of your base image has significantly fewer CVEs, which makes base image upgrades a concrete, measurable action rather than a vague best practice.

---

**"What is the difference between CMD and ENTRYPOINT?"**

Both define what runs when the container starts, but they serve different roles.

`CMD` defines the **default command**. It can be completely replaced by anything passed after the image name in `docker run`. If you run `docker run myimage bash`, the `CMD` in the Dockerfile is entirely ignored.

`ENTRYPOINT` defines the **fixed executable**. Arguments passed at `docker run` become arguments to the entrypoint — they do not replace it. This makes `ENTRYPOINT` appropriate when the container represents a specific tool that should always be run.

The most robust pattern combines both: `ENTRYPOINT` specifies the executable, `CMD` specifies the default arguments which the user can override:

```dockerfile
ENTRYPOINT ["python", "app.py"]
CMD ["--host", "0.0.0.0", "--port", "8080"]
```

Running `docker run myimage --port 9090` would execute `python app.py --port 9090`.

---

**"How do you clean up Docker resources? What are dangling images?"**

Dangling images are image layers that have no tag pointing to them. They accumulate when you rebuild an image with the same name and tag — the old layers lose their tag reference but remain on disk.

```bash
# List dangling images
docker images -f dangling=true

# Remove dangling images
docker image prune

# More aggressive: remove all unused images (not just dangling)
docker image prune -a

# Full cleanup: stopped containers, unused networks, dangling images, build cache
docker system prune

# Check disk usage by category
docker system df
```

In CI environments, build caches and images accumulate quickly. I run `docker system prune --volumes -f` in cleanup jobs on CI agents on a schedule to prevent disk exhaustion on build nodes.

---

## 12. Quick Reference Cheat Sheet

### Build

| Command | Purpose |
|---|---|
| `docker build -t myapp:1.0 .` | Build image from Dockerfile in current directory |
| `docker build --no-cache -t myapp:1.0 .` | Force rebuild all layers |
| `docker build --target build-stage .` | Build up to a specific multi-stage target |
| `DOCKER_BUILDKIT=1 docker build .` | Enable BuildKit for faster builds |
| `docker history myapp:1.0` | Show layers and sizes |

### Run & Manage Containers

| Command | Purpose |
|---|---|
| `docker run -d -p 8080:80 --name web nginx` | Run detached, map port, name the container |
| `docker run --rm -it myapp bash` | Run interactively, delete on exit |
| `docker run --memory=512m --cpus=1` | Apply resource limits |
| `docker run --read-only --tmpfs /tmp` | Read-only filesystem with tmpfs scratch |
| `docker run --cap-drop ALL --cap-add NET_BIND_SERVICE` | Drop capabilities |
| `docker exec -it web bash` | Shell into running container |
| `docker logs -f --tail=100 web` | Follow logs, last 100 lines |
| `docker stop --time=30 web` | Graceful stop with 30s grace period |
| `docker inspect web` | Full container JSON details |
| `docker stats` | Live CPU/memory/network usage |

### Images

| Command | Purpose |
|---|---|
| `docker images` | List local images |
| `docker pull nginx:alpine` | Pull image from registry |
| `docker tag myapp:latest registry.io/myapp:v1.2` | Tag for push |
| `docker push registry.io/myapp:v1.2` | Push to registry |
| `docker rmi myapp:old` | Remove image |
| `docker image prune` | Remove dangling images |
| `docker image prune -a` | Remove all unused images |
| `docker save myapp:1.0 | gzip > myapp.tar.gz` | Export image to file |
| `docker load < myapp.tar.gz` | Import image from file |

### Networking

| Command | Purpose |
|---|---|
| `docker network ls` | List all networks |
| `docker network create my-net` | Create user-defined bridge |
| `docker network inspect my-net` | Details: IPs, containers, config |
| `docker network connect my-net container1` | Connect container to network |
| `docker network disconnect my-net container1` | Disconnect |
| `docker run --network host nginx` | Use host network |
| `docker run --network none nginx` | No network access |

### Volumes

| Command | Purpose |
|---|---|
| `docker volume ls` | List volumes |
| `docker volume create my-vol` | Create named volume |
| `docker volume inspect my-vol` | Details including mount path |
| `docker volume prune` | Remove all unused volumes |
| `docker run -v my-vol:/data myapp` | Mount named volume |
| `docker run -v $(pwd)/config:/app/config:ro myapp` | Bind mount read-only |

### Inspect & Debug

| Command | Purpose |
|---|---|
| `docker inspect <id>` | Full JSON metadata |
| `docker inspect <id> --format='{{.State.OOMKilled}}'` | Check OOM status |
| `docker inspect <id> --format='{{.NetworkSettings.IPAddress}}'` | Container IP |
| `docker top <container>` | List processes inside container |
| `docker diff <container>` | Files changed in writable layer |
| `docker cp <container>:/app/log.txt ./` | Copy file out of container |

### Cleanup

| Command | Purpose |
|---|---|
| `docker ps -a` | List all containers (including stopped) |
| `docker rm $(docker ps -aq -f status=exited)` | Remove all stopped containers |
| `docker system prune` | Remove stopped containers, unused networks, dangling images |
| `docker system prune -a --volumes` | Aggressive full cleanup |
| `docker system df` | Disk usage by category |

### Compose

| Command | Purpose |
|---|---|
| `docker compose up -d` | Start all services in background |
| `docker compose down` | Stop and remove containers + networks |
| `docker compose down -v` | Also remove volumes (destructive!) |
| `docker compose logs -f service` | Follow logs for one service |
| `docker compose exec service bash` | Shell into running service |
| `docker compose ps` | Status of all services |
| `docker compose build --no-cache` | Force rebuild all service images |
| `docker compose pull` | Pull latest images |
