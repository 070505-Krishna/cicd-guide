# 18 — Glossary, Q&A, Exercises and Next Steps

## Presentation generator (`generate_presentation.py`)

A utility, **not part of the runtime pipeline**. It uses the `python-pptx` library to
build `CICD_Demo_Presentation.pptx` (16:9, 13.333 × 7.5 in) with a dark/light slide
theme. Run: `pip install python-pptx` then `python generate_presentation.py`.

Slides: (1) Title — *Enterprise CI/CD Automation*, (2) Project Overview & Engineering Objectives,
(3) End-to-End Pipeline Workflow, (4) Application Microservice Architecture (Flask),
(5) Containerization Strategy: Multi-Stage Docker, (6) Pipeline Orchestration: Jenkins as Code,
(7) Registry Distribution: Docker Hub, (8) Kubernetes Orchestration & Service Exposure,
(9) Verification, Smoke Testing & Rollback Safety, (10) Summary: what was accomplished + production enhancements.

Concepts: creating shapes/text boxes with `Inches`/`Pt` units, RGB colour palettes,
helper functions (`add_header`, `create_card`) to avoid repetition — "presentation as code".

## Glossary

| Term | Definition |
|------|-----------|
| **Agent / executor** | Jenkins worker / build slot |
| **Artifact** | Build output (here: a Docker image) |
| **ARG / ENV** | Build-time / runtime Dockerfile variables |
| **Build context** | Files sent to Docker for a build |
| **CD** | Continuous Delivery or Deployment |
| **CI** | Continuous Integration |
| **Cluster** | A set of machines running Kubernetes |
| **Container** | Running instance of an image |
| **Control loop** | Controller repeatedly reconciling actual → desired state |
| **CoreDNS** | Cluster DNS server |
| **Credentials binding** | Injecting secrets into a build step |
| **Crumb** | Jenkins CSRF token |
| **Declarative** | Describe the end state, not the steps |
| **Deployment** | K8s object managing ReplicaSets/rollouts |
| **Digest** | Immutable sha256 of an image |
| **DooD / DinD** | Docker-outside / inside-of-Docker |
| **Endpoint** | IP:port of a Ready pod behind a Service |
| **Gunicorn** | Production WSGI server |
| **HPA** | Horizontal Pod Autoscaler |
| **Idempotent** | Safe to repeat with the same result |
| **Ingress** | HTTP routing rules into a cluster |
| **JCasC** | Jenkins Configuration as Code |
| **Kubeconfig** | File telling kubectl how to reach/authenticate to a cluster |
| **kind** | Kubernetes IN Docker |
| **Label / selector** | Tag / query for K8s objects |
| **Liveness / readiness probe** | Restart check / traffic-readiness check |
| **Namespace** | Logical partition in a cluster |
| **NodePort** | Service type exposing a port on each node |
| **Orchestrator** | Software that schedules and manages containers |
| **Pod** | Smallest deployable K8s unit |
| **PAT** | Personal Access Token |
| **Registry** | Image storage server |
| **ReplicaSet** | Maintains N identical pods |
| **Rollback** | Return to previous version |
| **Rolling update** | Gradual pod replacement |
| **Smoke test** | Quick check that basics work after deployment |
| **Stage / step** | Pipeline phase / single action |
| **Tag** | Human-readable image version label |
| **WSGI** | Python web server ↔ app interface |

## Interview-style questions and answers

**1. Explain what happens from `git push` to a live update.**
Jenkins polls Git, checks out the commit, builds the Docker `test` stage (pytest), builds the `runtime` image
tagged `BUILD-SHA`, pushes to Docker Hub, renders the manifest with `sed`, applies it with `kubectl`, waits for the
rollout, runs a curl smoke test in a temporary pod, and rolls back if the build fails after deployment.

**2. Why a multi-stage Dockerfile?** One file gives a test gate and a lean, non-root production image without test tools; layers are shared via `base`.

**3. Why unique image tags instead of `latest`?** Traceability, immutability, reliable rollouts and rollbacks.

**4. How is zero downtime achieved?** `RollingUpdate` with `maxUnavailable: 0`, `maxSurge: 1`, 2 replicas and a readiness probe so traffic only goes to Ready pods.

**5. Readiness vs liveness?** Readiness controls receiving traffic; liveness controls restarting.

**6. How does the smoke test reach the app?** A temporary pod uses cluster DNS: `http://cicd-demo/health` → Service → pod.

**7. Why does Jenkins need `--internal` kubeconfig and `docker network connect kind jenkins`?** Because `127.0.0.1` inside the Jenkins container isn't the cluster; it must join the kind network and use the node's internal address.

**8. What are the security trade-offs of mounting `docker.sock`?** Gives root-equivalent control of the host's Docker; use rootless builders in production.

**9. What does `kubectl apply` do vs `create`?** `apply` is declarative/idempotent (create or update); `create` fails if the object exists.

**10. How do secrets flow?** `.env` → JCasC → Jenkins credential store → `withCredentials` → env var/temp file for a step → masked in logs.

**11. When does the automatic rollback not trigger?** If `rollout status` fails before `DEPLOYED` is set (see file 10 observation); K8s still keeps old pods serving.

**12. What is JCasC and why use it?** YAML-defined Jenkins config for reproducible, reviewable setups.

**13. Why exec-form `CMD`?** Process becomes PID 1 and receives SIGTERM for graceful shutdown.

**14. What does the HPA need to work?** metrics-server and CPU *requests* on the pods.

**15. What is the difference between Service `port`, `targetPort` and `nodePort`?** Service port / container port / node-level external port.

**16. How would you convert this to Continuous Delivery?** Add an `input` approval step before the Deploy stage.

**17. How would you move to GitOps?** Pipeline builds/pushes image and commits the new tag to a config repo; Argo CD/Flux pulls and applies it.

## Hands-on exercises (increasing difficulty)

1. **Run the local test**: `make local-test` — observe the test stage, runtime image and endpoint probes.
2. **Break a test** on purpose (change `"ok"` to `"fine"` in `app.py`); watch `docker build --target test` fail. Fix it.
3. **Inspect the image**: `docker history`, `docker run --rm cicd-demo:local id` (UID 10001).
4. **Add a `/ping` endpoint** + a test; run the pipeline and verify via `/version`.
5. **Change the greeting** and observe the rolling update with `kubectl get pods -w -n cicd-demo`.
6. **See load balancing**: loop `curl localhost:8081/version` and watch pod names alternate.
7. **Kill a pod** (`kubectl delete pod <name>`) and watch Kubernetes self-heal.
8. **Simulate a bad deploy**: make `/health` return 500; observe the stalled rollout and old pods still serving; then `kubectl rollout undo`.
9. **Scale**: `kubectl scale deployment/cicd-demo --replicas=4 -n cicd-demo`.
10. **Install metrics-server + apply `k8s/optional/hpa.yaml`**; generate load and watch scaling.
11. **Install ingress-nginx**, apply `ingress.yaml`, add `cicd-demo.local` to your hosts file.
12. **Fix the rollback gap**: set `env.DEPLOYED='true'` before `rollout status` and test a failing rollout.
13. **Add a lint stage** (`ruff`) and an image scan stage (`trivy image`).
14. **Replace `sed`** with Kustomize (`kustomize edit set image`) or `kubectl set image`.
15. **Replace polling with a GitHub webhook** using a tunnel (e.g., ngrok/cloudflared).
16. **Create a restricted ServiceAccount** + RBAC for Jenkins and swap it into the kubeconfig credential.
17. **Port the pipeline to GitHub Actions** to compare tools.

## Suggested next topics

* Helm charts & Kustomize overlays (dev/stage/prod)
* ConfigMaps and Secrets, persistent volumes, StatefulSets
* Observability: Prometheus, Grafana, Loki, OpenTelemetry
* GitOps with Argo CD
* Blue/green and canary with Argo Rollouts or a service mesh
* Terraform/OpenTofu for cluster provisioning
* Supply-chain security: SBOMs, cosign, SLSA

## Cheat sheet — most-used commands in this project

```bash
make local-test                              # quick validation, no cluster
make cluster                                 # kind + Jenkins
make deploy-local                            # deploy without Jenkins/registry
make status ; make logs ; make clean
kubectl -n cicd-demo get pods,svc
kubectl -n cicd-demo rollout undo deployment/cicd-demo
curl localhost:8081/version                  # which pod/version answered?
docker stop jenkins cicd-control-plane       # pause
docker start cicd-control-plane jenkins      # resume
```
