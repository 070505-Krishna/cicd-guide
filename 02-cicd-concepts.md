# 02 — CI/CD Concepts

## The problem CI/CD solves

Without automation, releasing software looks like this: a developer finishes a
feature, manually runs some tests (maybe), builds something on their laptop,
copies files to a server, restarts things, and hopes. This is slow, error-prone
and not repeatable ("it works on my machine").

**CI/CD** replaces manual steps with an automated, repeatable pipeline.

## Definitions

### Continuous Integration (CI)
Developers merge small changes into a shared branch frequently. Each change
automatically triggers **build + test**. Goal: find integration bugs within
minutes, not weeks.

In this project, CI = Jenkins stages **Checkout → Unit Tests → Build Image**.

### Continuous Delivery (CD)
Every change that passes CI is automatically packaged and made **ready** to
release. A human may still press the "deploy to production" button.

### Continuous Deployment (CD)
Every change that passes all automated checks is **deployed automatically** to
production, with no human approval.

This project is **Continuous Deployment** (to a local cluster): once tests pass
the pipeline pushes and deploys by itself.

| | CI | Continuous Delivery | Continuous Deployment |
|---|---|---|---|
| Builds & tests automatically | ✅ | ✅ | ✅ |
| Produces release-ready artifact | — | ✅ | ✅ |
| Human approval before prod | — | Usually yes | No |
| Deploys to prod automatically | — | No | ✅ |

## Anatomy of a pipeline

| Term | Meaning | Example here |
|------|---------|--------------|
| **Trigger** | What starts a run | `pollSCM('H/2 * * * *')`, or "Build Now" |
| **Pipeline** | The entire automated workflow | The `Jenkinsfile` |
| **Stage** | A logical phase | "Unit Tests", "Deploy to Kubernetes" |
| **Step** | One action inside a stage | `sh 'docker build ...'` |
| **Agent / executor** | Where the work runs | `agent any` → the Jenkins container |
| **Artifact** | Output worth keeping | Docker image `user/cicd-demo:12-ab12cd3` |
| **Environment** | Where the artifact runs | Namespace `cicd-demo` in kind |
| **Gate** | A check that can stop the flow | Failing pytest, failing smoke test |

## The stages in this project, and why each exists

| Stage | Question it answers |
|-------|---------------------|
| Checkout | "What exact code are we building?" |
| Unit Tests | "Does the code still behave correctly?" |
| Build Image | "Can we package it reproducibly?" |
| Push Image | "Can the cluster fetch exactly this build?" |
| Deploy | "Is the new version running?" |
| Smoke Test | "Is the running service actually healthy from inside the cluster?" |
| Post/rollback | "If not, can we restore the last good version?" |

## Principles of good pipelines

1. **Fail fast** — cheapest/fastest checks first (tests before pushing images).
2. **Build once, deploy many** — build an image once and promote *the same image* through environments.
3. **Everything as code** — pipeline, infrastructure, config in version control.
4. **Reproducible** — same input commit ⇒ same output image (pinned dependency versions like `Flask==3.0.3`).
5. **Traceable** — you can map a running pod back to a Git commit (tag `BUILD_NUMBER-GITSHA`).
6. **Idempotent** — running `kubectl apply` twice is safe.
7. **Observable** — timestamps, logs, health endpoints.
8. **Secure** — secrets never hardcoded in the repo.

## Deployment strategies (and where this project sits)

| Strategy | Idea | Downtime | Used here? |
|----------|------|----------|-----------|
| **Recreate** | Stop old, start new | Yes | No |
| **Rolling update** | Replace pods gradually | No (with probes) | **Yes** (`maxSurge 1`, `maxUnavailable 0`) |
| **Blue/Green** | Two full environments; switch traffic | No | No |
| **Canary** | Send small % of traffic to new version first | No | No |
| **Shadow** | Copy traffic to new version silently | No | No |

## Rollback strategies

* **Roll back the deployment** — `kubectl rollout undo` returns to the previous ReplicaSet (this project).
* **Roll forward** — ship a fix quickly instead of reverting.
* **Redeploy a previous image tag** — possible because every tag is unique and kept in the registry.

## Key metrics (DORA) that CI/CD improves

* **Deployment frequency** — how often you release.
* **Lead time for changes** — commit → production time.
* **Change failure rate** — % of deploys causing incidents.
* **Time to restore service** — how fast you recover (rollback helps here).

## Common CI/CD tools (for context)

| Category | Examples |
|----------|----------|
| CI servers | **Jenkins**, GitHub Actions, GitLab CI, CircleCI, Azure DevOps |
| Container build | Docker, BuildKit, Kaniko, Buildah |
| Registries | Docker Hub, GHCR, ECR, GCR/Artifact Registry, Harbor |
| Orchestrators | **Kubernetes**, Nomad, ECS |
| GitOps | Argo CD, Flux (pull-based deploys, an alternative to Jenkins running `kubectl`) |

## Push-based vs pull-based deployment

This project is **push-based**: Jenkins holds a kubeconfig and *pushes* changes
into the cluster. **GitOps** is **pull-based**: an in-cluster agent watches Git
and applies changes itself, so CI never needs cluster credentials. A common
upgrade path for this project would be: Jenkins builds + pushes the image and
updates a manifest in Git; Argo CD deploys.
