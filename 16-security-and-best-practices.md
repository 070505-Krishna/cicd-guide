# 16 — Security and Best Practices

Security in CI/CD ("DevSecOps") means protecting **code, secrets, build
infrastructure, artifacts and runtime**. This file shows what the project already
does well, what is a demo shortcut, and what to change for production.

## What the project already does well

| Practice | Where |
|----------|-------|
| Secrets kept out of Git | `.gitignore` excludes `.env`, `jenkins/.env`, `kubeconfig` |
| Secrets kept out of images | `.dockerignore` excludes `.env`, `kubeconfig`, `.git` |
| Template for secrets | `jenkins/.env.example` (no real values) |
| Credentials via Jenkins store | `withCredentials` + masking in logs |
| Token via stdin | `docker login --password-stdin` |
| Logout after use | `post { always { docker logout } }` |
| Non-root container | `USER 10001` in the runtime stage |
| No test tooling in prod image | Multi-stage build |
| Pinned dependencies | `Flask==3.0.3`, `gunicorn==22.0.0`, `pytest==8.3.2`, `curlimages/curl:8.8.0` |
| Resource limits | Deployment `limits` |
| Unique immutable tags | `BUILD-SHA` |
| No anonymous Jenkins access | `allowAnonymousRead: false`, no signup |
| Minimal pipeline blast radius | `timeout`, `disableConcurrentBuilds` |

## Demo shortcuts you should know are risky

| Shortcut | Risk | Better approach |
|----------|------|-----------------|
| Jenkins runs as **root** with **docker.sock** mounted | Any job can control the host Docker (≈ root on the host) | Kaniko/Buildah/BuildKit rootless on ephemeral agents |
| Default password `admin123` | Trivial to guess | Strong secret, SSO, rotate |
| Kubeconfig from kind = **cluster-admin** | Compromised Jenkins ⇒ compromised cluster | ServiceAccount + RBAC limited to namespace `cicd-demo` and verbs needed (`get/list/patch deployments`…) |
| `latest` tag pushed | Mutable tag | Deploy by unique tag/digest only |
| `loggedInUsersCanDoAnything` | Everyone is admin | Matrix/role-based authorization |
| Jenkins plugins unpinned (`jenkins-plugin-cli --plugins git …`) | Surprise updates | Pin versions (`git:5.2.2`) |
| `kubectl` fetched at build from "stable" | Not reproducible | Pin version and verify checksum |
| Docker Hub PAT with *Delete* | More privilege than needed | Read/Write only |
| Public repo/HTTP only | Eavesdropping | TLS everywhere |

## Container security checklist

* Run as non-root with a fixed UID ✅ (already).
* Add Kubernetes `securityContext`:
  ```yaml
  securityContext:
    runAsNonRoot: true
    runAsUser: 10001
    allowPrivilegeEscalation: false
    readOnlyRootFilesystem: true
    capabilities: { drop: ["ALL"] }
    seccompProfile: { type: RuntimeDefault }
  ```
* Use minimal base images (slim/distroless), rebuild regularly for patches.
* Scan images (Trivy/Grype) in the pipeline; fail on critical CVEs.
* Sign images (cosign) and enforce with admission policy.
* Never bake secrets into layers; use runtime Secrets.

## Kubernetes security checklist

* RBAC with least privilege; separate ServiceAccounts per app.
* NetworkPolicies (default-deny, then allow).
* Kubernetes **Secrets** (ideally encrypted at rest/external secrets manager) — note: base64 is *encoding*, not encryption.
* Pod Security Standards ("restricted" profile).
* Resource quotas/limits to prevent noisy-neighbour DoS.
* Keep the API server private; audit logs.

## Pipeline security

* **Branch protection** + code review before merge to `main` (since `main` auto-deploys).
* **Don't run untrusted PR code** with credentials (pipeline injection risk).
* Prefer single-quoted `sh '...'` so the shell, not Groovy, expands secrets ✅ (the Jenkinsfile does).
* Avoid printing env vars; masking is best-effort.
* Backups of `jenkins_home`; patch Jenkins core and plugins (frequent CVEs).
* Software supply chain: dependency scanning (pip-audit, Dependabot), SBOM generation, pinned base images.

## Secret handling cheat-sheet

| Secret | Stored | Used by |
|--------|--------|---------|
| Docker Hub PAT | `jenkins/.env` → Jenkins credential `dockerhub-creds` | Push stage |
| Kubeconfig | generated file, git-ignored → credential `kubeconfig` | Deploy, smoke test, rollback |
| Jenkins admin password | `jenkins/.env` | Login + REST calls |

If a secret leaks: **revoke/rotate immediately** (deleting the commit isn't enough — it stays in history and forks/clones).

## 12-Factor principles seen here

* **Config in environment** (`APP_VERSION`).
* **Logs to stdout** (`PYTHONUNBUFFERED=1`) so the platform collects them.
* **Stateless processes** — any pod can serve any request; easy to scale/replace.
* **Disposability** — fast start, graceful stop (exec-form CMD receives SIGTERM).
* **Dev/prod parity** — same image in tests and production.
