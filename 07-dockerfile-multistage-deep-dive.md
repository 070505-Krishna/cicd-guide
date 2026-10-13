# 07 — The Multi-Stage Dockerfile and `.dockerignore`

## The file (24 lines)

```dockerfile
# ---------- base: runtime dependencies ----------
FROM python:3.12-slim AS base
WORKDIR /app
ENV PYTHONDONTWRITEBYTECODE=1 PYTHONUNBUFFERED=1
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# ---------- test: runs unit tests; build fails if they fail ----------
FROM base AS test
COPY requirements-dev.txt pytest.ini ./
RUN pip install --no-cache-dir -r requirements-dev.txt
COPY app ./app
COPY tests ./tests
RUN pytest -q

# ---------- runtime: small, non-root production image ----------
FROM base AS runtime
ARG APP_VERSION=dev
ENV APP_VERSION=${APP_VERSION}
RUN useradd --system --uid 10001 appuser
COPY app ./app
USER 10001
EXPOSE 5000
CMD ["gunicorn", "-b", "0.0.0.0:5000", "-w", "2", "app.app:app"]
```

## What "multi-stage" means

A Dockerfile can contain several `FROM` lines. Each starts a **stage**. Stages can
build on one another (`FROM base AS test`) and you can choose which one to build
with `--target`.

```
                 ┌──────────► test    (adds pytest, copies tests, RUNS pytest)
base (python+deps)
                 └──────────► runtime (adds non-root user, app code, gunicorn CMD)
```

Why it's valuable:

* **One Dockerfile, two purposes.** `--target test` verifies the code; `--target runtime` produces the shippable image.
* **Smaller, safer runtime image.** pytest and test files exist **only** in the `test` stage; they never reach production.
* **Shared cache.** Both stages reuse the `base` layers (Python + Flask + gunicorn).
* **Tests are a build gate.** `RUN pytest -q` fails → build fails.

Note that `runtime` does **not** depend on `test`. Jenkins builds `test` first as
a gate and *then* builds `runtime` separately. (If you built only `runtime`,
tests would be skipped; with BuildKit unused stages may not even execute.)

## Line-by-line

### Base stage
| Line | Explanation |
|------|-------------|
| `FROM python:3.12-slim AS base` | Starts from the official Python 3.12 image, **slim** variant (Debian without extras; smaller than full, easier than Alpine because it uses glibc). Named `base`. |
| `WORKDIR /app` | Sets (and creates) the working directory; later `COPY`/`RUN`/`CMD` are relative to it. |
| `ENV PYTHONDONTWRITEBYTECODE=1` | Don't write `.pyc` files (cleaner image). |
| `ENV PYTHONUNBUFFERED=1` | Don't buffer stdout/stderr → logs appear immediately in `docker logs` / `kubectl logs`. |
| `COPY requirements.txt .` | Copy only the dependency list first (cache optimization). |
| `RUN pip install --no-cache-dir -r requirements.txt` | Install Flask + gunicorn. `--no-cache-dir` skips pip's download cache to keep the layer small. |

### Test stage
| Line | Explanation |
|------|-------------|
| `FROM base AS test` | Inherit everything from `base`. |
| `COPY requirements-dev.txt pytest.ini ./` | Dev dependency list and pytest config. |
| `RUN pip install -r requirements-dev.txt` | Adds pytest. |
| `COPY app ./app` / `COPY tests ./tests` | The code under test and the tests. |
| `RUN pytest -q` | **The gate.** Non-zero exit aborts the build. |

### Runtime stage
| Line | Explanation |
|------|-------------|
| `FROM base AS runtime` | Fresh from `base` — no pytest, no tests. |
| `ARG APP_VERSION=dev` | Build-time variable with default `dev`. |
| `ENV APP_VERSION=${APP_VERSION}` | Persist it as a runtime env var the Flask app reads. |
| `RUN useradd --system --uid 10001 appuser` | Create an unprivileged system user with a fixed numeric UID. |
| `COPY app ./app` | Only application code. |
| `USER 10001` | Run as that user, **not root**. Numeric UID is used so Kubernetes' `runAsNonRoot` checks can verify it. |
| `EXPOSE 5000` | Documents the listening port (metadata only). |
| `CMD [...]` | Default process. **Exec form** (JSON array) runs gunicorn as PID 1 directly, so it receives `SIGTERM` for graceful shutdown. Shell form would wrap it in `/bin/sh -c`. |

## `CMD` vs `ENTRYPOINT`, exec vs shell form

* `CMD` = default command, easily overridden at `docker run image <cmd>`.
* `ENTRYPOINT` = fixed executable; `CMD` becomes its default args.
* **Exec form** `["a","b"]` — no shell; signals reach your process. Preferred.
* **Shell form** `a b` — runs via `sh -c`; signals may not propagate.

## `COPY` vs `ADD`
Prefer `COPY`. `ADD` also auto-extracts archives and fetches URLs, which is surprising.

## `.dockerignore`

```
.git  __pycache__  *.pyc  .pytest_cache
kubeconfig  .env
jenkins  k8s  kind  scripts  docs
Makefile  README.md  Jenkinsfile
```

When you run `docker build .`, the whole directory (the **build context**) is sent
to the daemon. `.dockerignore` filters it, giving you:

1. **Speed** — a smaller context to upload.
2. **Cache stability** — editing the README won't invalidate layers.
3. **Security** — `kubeconfig` and `.env` can never be accidentally `COPY`'d into an image.
4. **Smaller images** — no `.git` history inside.

Note `tests/` and `pytest.ini` are intentionally **not** ignored — the test stage needs them.

## Build arguments passed by Jenkins

```
docker build --target runtime --build-arg APP_VERSION=${IMAGE_TAG} \
   -t ${IMAGE_NAME}:${IMAGE_TAG} -t ${IMAGE_NAME}:latest .
```
Two `-t` flags give **one image two tags**: the unique version tag and `latest`.

## Image size and security tips (further improvements)

* Pin the base image by digest (`python:3.12-slim@sha256:...`).
* Add a `HEALTHCHECK` instruction (Kubernetes ignores it but plain Docker uses it).
* Scan with **Trivy** or **Grype** for CVEs.
* Consider distroless/`chainguard` bases.
* Use BuildKit cache mounts: `RUN --mount=type=cache,target=/root/.cache/pip ...`.

## Inspecting what you built

```bash
docker history cicd-demo:local        # see layers and sizes
docker inspect cicd-demo:local        # full metadata
docker run --rm cicd-demo:local id    # shows uid=10001 (proves non-root)
docker run --rm cicd-demo:local env | grep APP_VERSION
```
