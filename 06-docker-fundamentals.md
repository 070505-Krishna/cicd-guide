# 06 — Docker Fundamentals

## What Docker is

Docker packages an application **together with its dependencies** into a
standard unit that runs the same everywhere. It uses Linux kernel features
(**namespaces** for isolation, **cgroups** for resource limits) rather than a
full virtual machine.

## Containers vs virtual machines

| | VM | Container |
|---|----|-----------|
| Includes | Full guest OS + kernel | App + libs; **shares host kernel** |
| Size | GBs | MBs |
| Start time | Minutes | Seconds/milliseconds |
| Isolation | Strong (hypervisor) | Process-level (namespaces/cgroups) |

## Core vocabulary

| Term | Meaning |
|------|---------|
| **Image** | Read-only template (layers + metadata). Built from a Dockerfile |
| **Container** | A running (or stopped) instance of an image |
| **Layer** | One filesystem diff produced by one Dockerfile instruction |
| **Registry** | Server storing images (Docker Hub) |
| **Repository** | Collection of tags for one image name (`user/cicd-demo`) |
| **Tag** | Human label for an image version (`:latest`, `:12-ab12cd3`) |
| **Digest** | Immutable content hash `sha256:...` of an image |
| **Build context** | Files sent to the Docker daemon for a build (the `.` in `docker build .`) |
| **Daemon** | `dockerd`, the background service doing the work |
| **CLI** | The `docker` command that talks to the daemon |

Architecture: `docker CLI ──(REST over socket)──► dockerd ──► containerd ──► runc ──► container`.

## Layers and caching

Each `RUN`, `COPY`, `ADD` creates a layer. Docker caches layers; if an
instruction and everything before it is unchanged, the cached layer is reused.

**Order matters.** The project's Dockerfile copies `requirements.txt` and runs
`pip install` **before** copying application code:

```
COPY requirements.txt .
RUN pip install ...        ← cached unless requirements.txt changes
COPY app ./app             ← changes often, so it comes later
```

So a code-only change rebuilds in seconds, not minutes.

## Commands used in this project

| Command | Meaning |
|---------|---------|
| `docker build --target test -t cicd-demo:test .` | Build only up to the `test` stage and tag it |
| `docker build --target runtime --build-arg APP_VERSION=1.0 -t name:tag .` | Build the runtime stage, pass a build argument |
| `docker run -d --name cicd-demo-smoke -p 5000:5000 cicd-demo:local` | Run detached (`-d`), named, publish host:container port |
| `docker rm -f cicd-demo-smoke` | Force-remove a container |
| `docker login -u USER --password-stdin` | Authenticate to Docker Hub |
| `docker push name:tag` | Upload image to the registry |
| `docker rmi name:tag` | Remove an image locally |
| `docker logout` | Remove stored credentials |
| `docker stop jenkins cicd-control-plane` / `docker start ...` | Pause / resume containers |
| `docker network connect kind jenkins` | Attach container `jenkins` to network `kind` |
| `docker compose ... up -d --build` | Start services defined in compose files |
| `docker compose ... down -v` | Stop and remove services **and volumes** |
| `docker exec jenkins cat ...` | Run a command inside a running container |

### Port publishing: `-p 5000:5000`
Format `HOST_PORT:CONTAINER_PORT`. `EXPOSE 5000` in a Dockerfile is only
documentation; `-p` actually opens the port on the host.

## Docker networking

* **bridge** (default): containers get private IPs on a virtual network and can
  reach each other by container name on *user-defined* bridges.
* **host**: share the host's network stack.
* **Named networks**: kind creates a network called `kind`. This project runs
  `docker network connect kind jenkins` so the Jenkins container can reach the
  Kubernetes API server container by name/IP.

## Volumes and bind mounts

| Type | Example in project | Purpose |
|------|--------------------|---------|
| Named volume | `jenkins_home:/var/jenkins_home` | Persist Jenkins data across container recreation |
| Bind mount | `./casc/jenkins.yaml:/casc/jenkins.yaml:ro` | Inject a host file; `:ro` = read-only |
| Socket mount | `/var/run/docker.sock:/var/run/docker.sock` | See next section |

`docker compose down -v` deletes named volumes too, so Jenkins state is wiped — that's why teardown uses `-v`.

## The Docker socket pattern ("Docker-outside-of-Docker")

Jenkins runs *in a container* but must run `docker build`. Solutions:

| Pattern | How | Trade-off |
|---------|-----|-----------|
| **DooD** (this project) | Mount host `/var/run/docker.sock`; install only the Docker **CLI** in Jenkins | Simple, fast, shares host image cache. **Gives Jenkins root-equivalent control over the host Docker** |
| DinD | Run a Docker daemon inside the container | Needs privileged mode, slower |
| Rootless builders (Kaniko, Buildah) | Build without a daemon | Best for production CI on Kubernetes |

That is why `docker-compose.yml` says `user: root # demo only`. Anyone who can
run `docker` through that socket can mount the host filesystem. Acceptable for a
laptop demo, not for production.

Consequence of DooD: images Jenkins builds appear in **your host's Docker**, and
containers it starts are **siblings**, not children.

## Environment variables, ARG vs ENV

| | `ARG` | `ENV` |
|---|---|---|
| Available | Only during build | During build **and** at runtime |
| Set with | `--build-arg NAME=value` | `ENV NAME=value` in Dockerfile, or `-e` at run |
| Project use | `ARG APP_VERSION=dev` | `ENV APP_VERSION=${APP_VERSION}` so the app can read it |

## Resource isolation

Docker (and Kubernetes) uses **cgroups** to enforce CPU/memory limits; in
Kubernetes these come from `resources.limits` (see file 13).

## Cleaning up

```bash
docker ps -a                 # all containers
docker images                # all images
docker system df             # disk usage
docker system prune          # remove unused data (careful!)
```
