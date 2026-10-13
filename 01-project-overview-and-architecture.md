# 01 — Project Overview and Architecture

## What this project is

A **fully automated, production-style CI/CD pipeline** running entirely on a
laptop. Every `git push` triggers this chain:

```
git push ──► Jenkins ──► Unit tests ──► Docker build ──► Docker Hub ──► kubectl apply ──► Kubernetes (2 pods)
               │        (in Docker)                                          │
               └────────── rollback on failure ◄──── smoke test ◄────────────┘
```

In words:

1. A developer pushes code to GitHub.
2. **Jenkins** notices the change (it polls Git roughly every 2 minutes).
3. Jenkins runs **unit tests inside Docker** (so tests run in a clean, reproducible environment).
4. If tests pass, Jenkins builds a **small production Docker image**.
5. Jenkins **pushes** the image to **Docker Hub**.
6. Jenkins **deploys** to a local **Kubernetes** cluster (created by **kind**) using `kubectl apply`.
7. Kubernetes performs a **rolling update** with zero downtime across **2 replicas (pods)**.
8. Jenkins runs an **in-cluster smoke test** (a temporary pod calls `/health`).
9. If something fails after deployment, Jenkins runs **`kubectl rollout undo`**.

## Technology stack

| Layer | Tool | Role in this project |
|-------|------|----------------------|
| Source control | Git + GitHub | Stores code; the trigger for the pipeline |
| Application | Python 3.12 + Flask 3.0.3 | A tiny web microservice |
| App server | gunicorn 22.0.0 | Production-grade WSGI server |
| Testing | pytest 8.3.2 | Unit tests for the endpoints |
| Packaging | Docker (multi-stage) | Builds test and runtime images |
| Registry | Docker Hub | Stores versioned images |
| CI/CD engine | Jenkins LTS (JDK 17) | Orchestrates the whole pipeline |
| Jenkins config | JCasC (Configuration as Code) | Auto-creates user, credentials, job |
| Orchestrator | Kubernetes | Runs and heals the app |
| Local cluster | kind (Kubernetes IN Docker) | A real cluster inside Docker containers |
| Automation | Bash scripts + Makefile | One-command setup, test, teardown |
| Docs | python-pptx script | Generates a 10-slide presentation |

## File and folder map

| Path | Purpose | Explained in |
|------|---------|--------------|
| `app/app.py` | Flask service with `/`, `/health`, `/version` | 04 |
| `app/__init__.py` | Empty file; makes `app` a Python package | 04 |
| `tests/test_app.py` | 4 pytest tests | 05 |
| `pytest.ini` | Tells pytest where tests are and sets `pythonpath` | 05 |
| `requirements.txt` | Runtime deps: Flask, gunicorn | 04, 07 |
| `requirements-dev.txt` | Adds pytest (and includes `requirements.txt`) | 05, 07 |
| `Dockerfile` | Multi-stage: `base` → `test` and `runtime` | 07 |
| `.dockerignore` | Keeps junk/secrets out of the build context | 07 |
| `.gitignore` | Keeps junk/secrets out of Git | 03 |
| `Jenkinsfile` | The pipeline definition (Declarative Pipeline) | 10 |
| `jenkins/Dockerfile` | Custom Jenkins image with Docker CLI + kubectl + plugins | 11 |
| `jenkins/docker-compose.yml` | Runs Jenkins container | 11 |
| `jenkins/docker-compose.casc.yml` | Override that enables JCasC | 11 |
| `jenkins/casc/jenkins.yaml` | JCasC: admin user, credentials, pipeline job | 11 |
| `jenkins/.env.example` | Template for secrets file `jenkins/.env` | 11, 16 |
| `k8s/namespace.yaml` | Creates namespace `cicd-demo` | 13 |
| `k8s/deployment.yaml` | 2 replicas, rolling update, probes, limits | 13 |
| `k8s/service.yaml` | NodePort Service (80 → 5000, nodePort 30080) | 13 |
| `k8s/optional/hpa.yaml` | Autoscaler (2–5 pods at 60% CPU) | 13 |
| `k8s/optional/ingress.yaml` | Ingress host `cicd-demo.local` | 13 |
| `kind/kind-config.yaml` | Maps container port 30080 → host 8081 | 14 |
| `scripts/setup.sh` | One-command cluster + Jenkins setup | 15 |
| `scripts/local-test.sh` | Test + build + probe without Jenkins/K8s | 15 |
| `scripts/teardown.sh` | Cleans up everything | 15 |
| `Makefile` | Shortcut targets (`make test`, `make cluster`, ...) | 15 |
| `docs/TROUBLESHOOTING.md` | Symptom/fix table | 17 |
| `generate_presentation.py` | Builds `CICD_Demo_Presentation.pptx` | 18 |

## Network and port map (very important to understand)

```
Your browser ──► localhost:8081
                     │   (kind extraPortMappings: host 8081 → node container 30080)
                     ▼
            kind node container (cicd-control-plane)
                     │   NodePort 30080
                     ▼
            Service "cicd-demo"  (port 80)
                     │   selects pods with label app=cicd-demo
                     ▼
            Pod 1 :5000   /   Pod 2 :5000   (gunicorn + Flask)

Jenkins UI ──► localhost:8080   (container "jenkins")
Jenkins agent port ─► 50000
Jenkins ──► Docker daemon via /var/run/docker.sock (host's Docker)
Jenkins ──► Kubernetes API via kubeconfig, over the Docker network "kind"
```

## Where each "environment" lives

* **Developer machine**: your code editor and Git.
* **Docker Desktop (host Docker engine)**: runs Jenkins container AND the kind node container, and builds images.
* **kind cluster**: a Kubernetes cluster whose "node" is itself a Docker container named `cicd-control-plane`.
* **Docker Hub**: the remote registry; the only thing outside your machine.

## Key design ideas demonstrated

* **Pipeline as code** — the pipeline lives in the `Jenkinsfile` inside the repo.
* **Infrastructure as code** — Jenkins config (JCasC) and Kubernetes manifests are text files in Git.
* **Immutable, traceable artifacts** — each image is tagged `BUILD_NUMBER-GITSHA`.
* **Shift-left testing** — tests run *before* anything is built for production.
* **Smaller, safer images** — multi-stage build, non-root user, no test tools in runtime image.
* **Zero-downtime deploys** — `maxUnavailable: 0` + readiness probes.
* **Self-healing** — liveness probes + Deployment controller.
* **Automated verification and rollback** — smoke test + `rollout undo`.
