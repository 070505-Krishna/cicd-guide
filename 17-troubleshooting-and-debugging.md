# 17 — Troubleshooting and Debugging

Source: `docs/TROUBLESHOOTING.md`. Each row is explained here with the **why**
and the commands to **confirm** the diagnosis.

## A systematic method (works for any CI/CD problem)

1. **Locate the failing stage** (Jenkins "Stage View" / console log).
2. **Read the first error**, not the last — later errors are often consequences.
3. **Reproduce locally** with the same command (`make test`, `make local-test`, `kubectl …`).
4. **Check the layer boundaries**: Git → Jenkins → Docker → Registry → Kubernetes → Network.
5. **Change one thing at a time.**

```
Where did it break?
 ├─ Checkout      → Git URL, branch, credentials
 ├─ Unit Tests    → real test failure or dependency problem (see docker build output)
 ├─ Build/Push    → Docker daemon access, Docker Hub login/permissions
 ├─ Deploy        → kubeconfig / network / image pull / manifest errors
 └─ Smoke test    → app not healthy, Service/selector, DNS
```

## Symptom table, explained

### 1. `permission denied ... /var/run/docker.sock` (in Jenkins)
* **Why**: the Docker CLI in Jenkins talks to the host daemon through the mounted socket, which only root/`docker` group can use.
* **Fix**: the Compose file runs Jenkins as `root` (demo). Recreate: `docker compose up -d --force-recreate`.
* **Verify**: `docker exec jenkins docker ps` should list containers.

### 2. `Unable to connect to the server` / `dial tcp 127.0.0.1` in the Deploy stage
* **Why**: kubeconfig points at `127.0.0.1`, which inside the Jenkins container is Jenkins itself.
* **Fix**:
  ```bash
  kind get kubeconfig --internal --name cicd > kubeconfig
  # re-upload/refresh the 'kubeconfig' credential
  docker network connect kind jenkins
  ```
* **Verify**: `docker exec jenkins kubectl --kubeconfig /casc/kubeconfig get nodes` (when using JCasC mount).

### 3. `ImagePullBackOff` / `ErrImagePull`
* **Why**: the node can't download the image — private repo, wrong `DOCKERHUB_USER`, typo in tag, or no internet.
* **Fix**: make the Docker Hub repo public, or create an `imagePullSecret`; correct the username; or bypass the registry with `make deploy-local`.
* **Verify**: `kubectl -n cicd-demo describe pod -l app=cicd-demo` → read **Events** (they quote the exact pull error).
* *Back-off* means Kubernetes retries with increasing delays.

### 4. `denied: requested access to the resource is denied` on push
* **Why**: wrong username/token, token lacks write scope, or `DOCKERHUB_USER` in the Jenkinsfile is someone else's namespace (**especially after forking!** the file still has the original author's name).
* **Fix**: update the Jenkinsfile and the `dockerhub-creds` credential; PAT with Read/Write.
* **Verify locally**: `docker login -u YOUR_USER` then `docker push YOUR_USER/cicd-demo:test`.

### 5. Pipeline can't find credentials
* **Why**: IDs must be **exactly** `dockerhub-creds` and `kubeconfig` (case-sensitive).
* **Fix**: check *Manage Jenkins → Credentials*, or re-run JCasC.

### 6. `localhost:8081` not responding
* **Why**: the cluster wasn't created with `kind/kind-config.yaml`, so no host→node port mapping exists.
* **Fix**: recreate the cluster, or run `kubectl -n cicd-demo port-forward svc/cicd-demo 8081:80`.
* **Verify**: `docker ps` shows `0.0.0.0:8081->30080/tcp` on `cicd-control-plane`.

### 7. Rollout times out
* **Why**: new pods never become Ready — image pull failure, crash, or `/health` failing.
* **Fix/diagnose**:
  ```bash
  kubectl -n cicd-demo get pods
  kubectl -n cicd-demo describe pod -l app=cicd-demo
  kubectl -n cicd-demo logs -l app=cicd-demo --tail=50      # make logs
  kubectl -n cicd-demo rollout status deployment/cicd-demo
  ```
* Remember: with `maxUnavailable: 0`, old pods keep serving during a failed rollout.

### 8. Port 8080 already in use
* **Why**: another program uses it.
* **Fix**: change the **left** side of `"8080:8080"` in `docker-compose.yml` (host:container), e.g. `"9090:8080"`.

### 9. Apple Silicon / ARM
* Python, Jenkins and kubectl images/binaries are multi-arch (the Jenkins Dockerfile uses `dpkg --print-architecture`), so no changes are needed.

### 10. JCasC startup error
* **Why**: missing variable in `jenkins/.env`, or `kubeconfig` file isn't at the project root.
* **Fix**: `docker logs jenkins`; run `scripts/setup.sh` once; or delete `jenkins/.env` to fall back to manual setup.

## Extra problems you may meet

| Symptom | Likely cause | Fix |
|---------|--------------|-----|
| `CrashLoopBackOff` | App crashes at start (bad CMD, import error, OOM) | `kubectl logs --previous <pod>`; check limits |
| `OOMKilled` | Exceeded 128Mi memory limit | Raise limit or reduce workers |
| Service returns nothing / "no endpoints" | Selector doesn't match pod labels, or pods not Ready | `kubectl get endpoints -n cicd-demo` |
| `error: unable to recognize ... no matches for kind` | Wrong apiVersion/CRD missing | Check manifest header |
| `sed` leaves `DOCKER_IMAGE_PLACEHOLDER` | Applied raw `deployment.yaml` instead of rendered file | Use the rendered file or `make deploy-local` |
| Test stage fails with `ModuleNotFoundError: app` | `pytest.ini`/`__init__.py` missing or wrong build context | Keep `pythonpath = .`; check `.dockerignore` |
| Build uses stale code | Docker cache | `docker build --no-cache ...` |
| Stuck `Terminating` pods | Graceful shutdown/finalizers | `kubectl delete pod --force --grace-period=0` (last resort) |
| `kubectl` parse error on Windows | UTF-16/BOM kubeconfig | Re-create with UTF-8 no BOM (see README script) |
| Smoke pod name already exists | Leftover `smoke-N` pod | `kubectl -n cicd-demo delete pod smoke-N` |
| Jenkins build "stuck pending" | Another build running (`disableConcurrentBuilds`) | Wait/abort the other build |

## Debug toolbox

```bash
# Docker / Jenkins
docker ps -a ; docker logs -f jenkins ; docker exec -it jenkins bash
docker exec jenkins docker version ; docker exec jenkins kubectl version --client

# Kubernetes
kubectl get all -n cicd-demo
kubectl describe deployment cicd-demo -n cicd-demo
kubectl get events -n cicd-demo --sort-by=.lastTimestamp
kubectl exec -it <pod> -n cicd-demo -- sh        # runtime image has no curl; use python -c or a debug pod
kubectl run dbg --rm -it --image=curlimages/curl:8.8.0 -n cicd-demo -- sh
kubectl get deploy cicd-demo -n cicd-demo -o yaml

# Network
curl -v localhost:8081/health
docker network inspect kind
```

## Reading `kubectl describe pod` events

Look at the bottom **Events** section: `Scheduled → Pulling → Pulled → Created → Started`.
Anything with `Warning` (`Failed`, `BackOff`, `Unhealthy`) is your clue:
* `Readiness probe failed: HTTP probe failed with statuscode: 500` → app problem.
* `Back-off pulling image` → registry problem.
* `Liveness probe failed ... Killing` → restarts.
