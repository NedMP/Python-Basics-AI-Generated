

# Docker Automation with Python (`docker-py`) — updated Oct 6, 2025

Automate Docker from Python using the official **Docker SDK for Python** (`docker-py`). Build images, run containers, stream logs, exec commands, manage networks/volumes, and push/pull from registries **programmatically**. Use with `002_SETUP.md` for venv and `.env` handling. Works on macOS with Docker Desktop or against remote Linux hosts over TCP/SSH.

---

**Assumptions**  
- macOS + zsh, Python 3.12–3.13 in a venv.  
- Docker Desktop or a reachable Docker Engine.  
- Secrets loaded via `.env` (never hardcode registry creds).

**Install**
```sh
pip install docker python-dotenv
```
> Ensure Docker Desktop is running on macOS, or set `DOCKER_HOST` for remote engines.

---

## Table of Contents
- [0) Connect to Docker](#0-connect-to-docker)
- [1) Build images](#1-build-images)
- [2) Run containers](#2-run-containers)
- [3) Inspect, logs, exec, and stats](#3-inspect-logs-exec-and-stats)
- [4) Volumes and bind mounts](#4-volumes-and-bind-mounts)
- [5) Networks](#5-networks)
- [6) Registry auth, push, and pull](#6-registry-auth-push-and-pull)
- [7) Cleaning up: stop, remove, prune](#7-cleaning-up-stop-remove-prune)
- [8) Patterns for job runners and health checks](#8-patterns-for-job-runners-and-health-checks)
- [9) Remote engines over TCP/SSH](#9-remote-engines-over-tcpssh)
- [10) Error handling and timeouts](#10-error-handling-and-timeouts)
- [11) Troubleshooting](#11-troubleshooting)
- [12) Recap](#12-recap)

---

## 0) Connect to Docker
**What**: Create a client bound to your local or remote Docker Engine.  
**Why**: All SDK actions go through the client. Environment variables can switch targets without code changes.
```python
# file: dk_connect.py
from dotenv import load_dotenv; load_dotenv()
import docker, os

# Prefer environment configuration for portability
# e.g. DOCKER_HOST=tcp://10.0.0.5:2375 or ssh://user@host
# DOCKER_TLS_VERIFY=1, DOCKER_CERT_PATH=~/.docker

client = docker.from_env()
print(client.version()["Version"])  # engine version
```
If you need low‑level API access (streaming build logs, advanced flags):
```python
api = client.api  # docker.APIClient
```

---

## 1) Build images
**What**: Turn a Dockerfile + context folder into an image tag.  
**Why**: Reproducible environments and portable deployments.

### 1.1 Minimal Dockerfile
```dockerfile
# file: Dockerfile
FROM python:3.13-slim
WORKDIR /app
COPY app.py .
CMD ["python","app.py"]
```

### 1.2 Build via high‑level API
```python
# file: dk_build.py
import docker
client = docker.from_env()
img, logs = client.images.build(path=".", tag="py-notes:latest", rm=True, pull=True)
for l in logs:
    line = l.get('stream') or l.get('status')
    if line:
        print(line.strip())
print("built:", img.tags)
```

### 1.3 Low‑level streaming build (more control)
```python
# file: dk_build_stream.py
import docker, json
api = docker.APIClient.from_env()
for chunk in api.build(path='.', tag='py-notes:dev', decode=True, rm=True, pull=True):
    if 'stream' in chunk:
        print(chunk['stream'].strip())
    elif 'errorDetail' in chunk:
        raise RuntimeError(chunk['errorDetail']['message'])
```
Tips:
- Use **build args**: `client.images.build(buildargs={"VERSION":"1.2"}, ...)`.  
- Use **.dockerignore** to shrink context and speed up builds.

---

## 2) Run containers
**What**: Start containers with environment, ports, and mounts.  
**Why**: Programmatic control over runtime config and lifecycle.
```python
# file: dk_run.py
import docker
client = docker.from_env()

container = client.containers.run(
    image="python:3.13-slim",
    command=["python","-c","print('hello from container')"],
    environment={"ENV_NAME":"Ned"},
    name="hello-demo",
    detach=True,
    tty=True,
)
print("started:", container.id[:12])
print("status:", container.status)
```
Expose ports and bind mounts:
```python
container = client.containers.run(
    "nginx:alpine",
    ports={"80/tcp": 8080},                 # host 8080 → container 80
    volumes={"/host/site": {"bind": "/usr/share/nginx/html", "mode": "ro"}},
    detach=True,
    name="web1",
    restart_policy={"Name":"unless-stopped"},
)
```

---

## 3) Inspect, logs, exec, and stats
**What**: Observe and control running containers.  
**Why**: Needed for automation, health, and debugging.
```python
c = client.containers.get("web1")
print(c.status)
print(c.attrs["NetworkSettings"]["IPAddress"])  # inspect details

# logs (stream)
for line in c.logs(stream=True, tail=10):
    print(line.decode().rstrip())

# exec inside container
exit_code, output = c.exec_run(["sh","-lc","nginx -v"], demux=True)
print(exit_code, (output[0] or b"" + output[1] or b"").decode(errors="ignore"))

# stats (non-blocking sample)
print(c.stats(decode=True, stream=False))
```

---

## 4) Volumes and bind mounts
**What**: Persist data or share files with the host.  
**Why**: Keep state beyond container lifecycle.
```python
# named volume
vol = client.volumes.create(name="data1")
client.containers.run("redis:alpine", detach=True, name="cache1", volumes={"data1":{"bind":"/data","mode":"rw"}})

# bind mount host directory
client.containers.run("busybox", "sh -c 'ls -la /host'", volumes={"/Users/ned/tmp":{"bind":"/host","mode":"rw"}}, remove=True)
```

---

## 5) Networks
**What**: Isolate and connect services.  
**Why**: Service discovery and predictable addressing.
```python
net = client.networks.create("appnet", driver="bridge")
web = client.containers.run("nginx:alpine", name="webnet", detach=True, network=net.name)
api = client.containers.run("python:3.13-slim", name="apinet", detach=True, network=net.name, command=["sleep","3600"])

# connect/disconnect later
net.connect(api, aliases=["api"])  # container can be looked up by alias
net.disconnect(api)
```

---

## 6) Registry auth, push, and pull
**What**: Work with private registries.  
**Why**: CI/CD pipelines and mirrored images.
```python
from dotenv import load_dotenv; load_dotenv()
import os, docker
client = docker.from_env()

auth = {"username": os.getenv("REG_USER"), "password": os.getenv("REG_PASS")}
client.images.pull("registry.example.com/tools/worker:1.0", auth_config=auth)
client.images.push("registry.example.com/tools/worker:1.0", auth_config=auth)
```
For ECR/GCR/ACR, obtain a token (CLI or SDK) then pass to `auth_config`.

---

## 7) Cleaning up: stop, remove, prune
**What**: Keep hosts tidy and reclaim space.  
**Why**: Stale containers and layers waste disk and cause confusion.
```python
# stop & remove container
c = client.containers.get("web1")
c.stop(timeout=10)
c.remove(v=True)  # remove anonymous volumes

# remove image by tag
i = client.images.get("py-notes:latest")
client.images.remove(i.id, force=False, noprune=False)

# prune dangling images/containers/networks/volumes
client.containers.prune()
client.images.prune()
client.networks.prune()
client.volumes.prune()
```

---

## 8) Patterns for job runners and health checks
**What**: Run short jobs and verify health before proceeding.  
**Why**: Deterministic automation in CI and schedulers.
```python
# short job pattern
log = client.containers.run("alpine", ["sh","-lc","echo hi && exit 0"], remove=True)
print(log.decode())

# wait on healthcheck
import time
svc = client.containers.run("postgres:16-alpine", name="pg1", detach=True, healthcheck={
  "test": ["CMD-SHELL", "pg_isready -U postgres"],
  "interval": 5_000_000_000,   # 5s in ns
  "timeout": 2_000_000_000,
  "retries": 10
})

for _ in range(60):
    svc.reload()
    if svc.attrs.get("State",{}).get("Health",{}).get("Status") == "healthy":
        break
    time.sleep(1)
else:
    raise SystemExit("service not healthy in time")
```

---

## 9) Remote engines over TCP/SSH
**What**: Operate Docker on remote hosts.  
**Why**: Centralized builders or production servers.
```python
import docker
client = docker.DockerClient(base_url="ssh://ubuntu@prod-host")  # requires SSH agent keys
print(client.ping())
```
TCP with TLS example (certs under `~/.docker`):
```python
import docker, os
client = docker.DockerClient(base_url=os.getenv("DOCKER_HOST"))
```
Ensure daemon is configured to listen on TCP with TLS.

---

## 10) Error handling and timeouts
**What**: Fail fast with context and retry transient conditions.  
**Why**: Networks flake and images conflict.
```python
import docker
from docker.errors import APIError, NotFound, ImageNotFound, DockerException

client = docker.from_env(timeout=30)  # HTTP timeout
try:
    client.containers.get("missing").remove()
except NotFound:
    print("Container not found; ok")
except APIError as e:
    print("Docker API error:", e.explanation)
except DockerException as e:
    print("Client error:", e)
```
Guidelines:
- Set client timeouts; avoid indefinite waits.  
- Use **unique names** or labels to avoid collisions; filter by `filters={"label":"app=xyz"}`.  
- Capture build output for diagnostics.

---

## 11) Troubleshooting
- **`docker.errors.DockerException: Error while fetching server API version`**: Docker is not running or `DOCKER_HOST` wrong. Start Docker Desktop or fix env.  
- **Permission denied on socket**: add user to `docker` group on Linux or use rootless Docker.  
- **Bind mount path not found**: create host path first and use absolute paths.  
- **Port already allocated**: stop conflicting container or map to a different host port.  
- **Image pull rate limits/auth**: pass `auth_config` or log in with `docker login` and ensure credential helper works.

---

## 12) Recap
```plaintext
Create client → build image → run container with env/ports/mounts → inspect logs/exec → manage volumes/networks → push/pull with auth → clean up → handle errors and timeouts → target remote engines when needed
```

**Next**: Schedule builds and runs (`026_TASK_SCHEDULING.md`), store secrets in Keychain (`033_KEYCHAIN_SECRETS.md`), and push artifacts to registries as part of CI. For packaging your Python app into images, combine with `035_SCRIPT_PACKAGING.md`. 