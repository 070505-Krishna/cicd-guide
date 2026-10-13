# 10 — The Jenkinsfile, Line by Line

File: `Jenkinsfile` (109 lines, Declarative Pipeline). This is the heart of the project.

## Overall skeleton

```groovy
pipeline {
  agent any
  options { ... }
  triggers { ... }
  environment { ... }
  stages { stage('Checkout') {...}  stage('Unit Tests') {...}  stage('Build Image') {...}
           stage('Push Image') {...} stage('Deploy to Kubernetes') {...} stage('Smoke Test') {...} }
  post { failure {...}  always {...} }
}
```

```
Checkout → Unit Tests → Build Image → Push Image → Deploy → Smoke Test
                                                                │ fail
                         post.failure ◄─────────────────────────┘ (rollback if deployed)
                         post.always  (cleanup)
```

## `agent any`
Run on any available agent/executor — here, the Jenkins container itself.

## `options`

| Option | Meaning |
|--------|---------|
| `timestamps()` | Prefix every log line with a time (needs Timestamper plugin) |
| `disableConcurrentBuilds()` | Only one build of this job at a time; prevents two deploys racing each other |
| `timeout(time: 20, unit: 'MINUTES')` | Abort the build if it hangs more than 20 minutes |

## `triggers { pollSCM('H/2 * * * *') }`
Check Git roughly every 2 minutes; build if there are new commits. (See file 03.)

## `environment`

```groovy
DOCKERHUB_USER   = 'likhithkumarp07'          // ← must be YOUR Docker Hub name if you fork
IMAGE_NAME       = "${DOCKERHUB_USER}/cicd-demo"
REGISTRY_CREDS   = 'dockerhub-creds'          // credential ID
KUBECONFIG_CREDS = 'kubeconfig'               // credential ID
K8S_NAMESPACE    = 'cicd-demo'
```
These become environment variables for every `sh` step. Single quotes = literal
string; double quotes = Groovy **string interpolation** (`"${DOCKERHUB_USER}/cicd-demo"`).

## Stage 1 — Checkout

```groovy
checkout scm
script {
  env.GIT_SHORT = sh(returnStdout: true, script: 'git rev-parse --short HEAD').trim()
  env.IMAGE_TAG = "${env.BUILD_NUMBER}-${env.GIT_SHORT}"
  echo "Image tag: ${env.IMAGE_TAG}"
}
```
* `checkout scm` — clone/update the repo configured for the job at the triggering commit.
* `sh(returnStdout: true, ...)` — run a shell command and capture its output as a Groovy string; `.trim()` removes the newline.
* Assigning to `env.X` makes the value available to **later stages** as an environment variable.
* Result e.g. `IMAGE_TAG = 17-a1b2c3d`.

## Stage 2 — Unit Tests

```groovy
sh 'docker build --target test -t ${IMAGE_NAME}:test-${IMAGE_TAG} .'
```
Builds only the Dockerfile's `test` stage, which runs `pytest`. A test failure
makes `docker build` exit non-zero → **stage fails → pipeline stops** (nothing is
pushed or deployed). Tagged `…:test-17-a1b2c3d` so it's identifiable and removable.

## Stage 3 — Build Image

```bash
docker build --target runtime \
  --build-arg APP_VERSION=${IMAGE_TAG} \
  -t ${IMAGE_NAME}:${IMAGE_TAG} \
  -t ${IMAGE_NAME}:latest .
```
Produces the lean production image, bakes in the version string, and applies two tags.
`sh '''...'''` is a Groovy triple-single-quoted multi-line string; `\` continues the shell line.
Because it's single-quoted, **Groovy doesn't interpolate** — the *shell* expands
`${IMAGE_TAG}` from the environment (safer against injection and avoids secret leakage into Groovy).

## Stage 4 — Push Image

```groovy
withCredentials([usernamePassword(credentialsId: env.REGISTRY_CREDS,
                                  usernameVariable: 'REG_USER',
                                  passwordVariable: 'REG_PASS')]) {
  sh '''
    echo "$REG_PASS" | docker login -u "$REG_USER" --password-stdin
    docker push ${IMAGE_NAME}:${IMAGE_TAG}
    docker push ${IMAGE_NAME}:latest
  '''
}
```
* `withCredentials` injects `REG_USER` / `REG_PASS` only inside the block, masked in logs.
* Login via stdin, then push both tags.

## Stage 5 — Deploy to Kubernetes

```groovy
withCredentials([file(credentialsId: env.KUBECONFIG_CREDS, variable: 'KUBECONFIG')]) {
  sh '''
    sed "s|DOCKER_IMAGE_PLACEHOLDER|${IMAGE_NAME}:${IMAGE_TAG}|" k8s/deployment.yaml > k8s/deployment.rendered.yaml
    kubectl apply -f k8s/namespace.yaml
    kubectl apply -f k8s/deployment.rendered.yaml
    kubectl apply -f k8s/service.yaml
    kubectl -n ${K8S_NAMESPACE} rollout status deployment/cicd-demo --timeout=120s
  '''
  script { env.DEPLOYED = 'true' }
}
```

1. `file(...)` credential: Jenkins writes the secret kubeconfig to a temp file and
   sets `KUBECONFIG` to its path. `kubectl` automatically reads `$KUBECONFIG`.
2. **Manifest templating with `sed`**: replaces the literal placeholder
   `DOCKER_IMAGE_PLACEHOLDER` with the real image. `|` is used as the sed
   delimiter because the image contains `/`. The output is a *generated* file
   (`deployment.rendered.yaml`, git-ignored). The source manifest stays generic.
   *(Production-grade alternatives: Kustomize `images:`, Helm values, or `kubectl set image`.)*
3. **Apply order**: namespace first (others live inside it), then Deployment, then Service.
   `kubectl apply` is **declarative and idempotent**: "make the cluster match this file".
4. `rollout status ... --timeout=120s` blocks until the rollout is complete (new pods
   Ready, old pods gone) or times out with a failure.
5. `env.DEPLOYED = 'true'` records that a deploy happened, to be used in `post`.

## Stage 6 — Smoke Test

```bash
kubectl -n ${K8S_NAMESPACE} run smoke-${BUILD_NUMBER} --rm -i --restart=Never \
  --image=curlimages/curl:8.8.0 -- curl -fsS http://cicd-demo/health
```

| Piece | Meaning |
|-------|---------|
| `kubectl run smoke-17` | Create a one-off **pod** named per build |
| `--rm` | Delete the pod when it finishes |
| `-i` | Attach so output returns and the exit code propagates |
| `--restart=Never` | Plain pod (not a Deployment), runs once |
| `--image=curlimages/curl:8.8.0` | Tiny image containing `curl` (pinned version) |
| `curl -fsS` | `-f` fail on HTTP ≥ 400 (exit non-zero), `-s` silent, `-S` still show errors |
| `http://cicd-demo/health` | The **Service DNS name** (short form inside the same namespace) on port 80 |

This tests the **real path** inside the cluster: DNS → Service → kube-proxy → a pod,
rather than just trusting "the pods are Running".

## `post` section

```groovy
post {
  failure {
    script {
      if (env.DEPLOYED == 'true') {
        echo 'Deployment unhealthy - rolling back'
        withCredentials([file(...)]) {
          sh 'kubectl -n ${K8S_NAMESPACE} rollout undo deployment/cicd-demo || true'
        }
      }
    }
  }
  always {
    sh 'docker logout || true'
    sh 'docker rmi ${IMAGE_NAME}:test-${IMAGE_TAG} || true'
  }
}
```

* `post` runs after stages. `failure` = only if the build failed. `always` = regardless.
* **Rollback**: `kubectl rollout undo` switches the Deployment back to the previous
  ReplicaSet revision (kept thanks to `revisionHistoryLimit: 5`).
* `|| true` means "don't let this command's own failure change the outcome" (e.g.,
  there's no previous revision on the very first deploy, or the image doesn't exist).
* **Cleanup**: `docker logout` removes the stored token from the Jenkins container's
  Docker config; `docker rmi` removes the throwaway test image.

> ### Observation — when does the automatic rollback actually fire?
> `DEPLOYED` is set **after** `rollout status` succeeds. Therefore:
> * **Smoke test fails** → `DEPLOYED == 'true'` → `rollout undo` runs. ✅
> * **`rollout status` times out** (e.g., new pods never become Ready) →
>   the `sh` step throws before `DEPLOYED` is set → **no explicit undo**.
>   In that case Kubernetes itself still protects users: with `maxUnavailable: 0`
>   and a failing readiness probe, the **old pods keep serving traffic** and the
>   new ReplicaSet never becomes available. The build is red, and you'd inspect with
>   `kubectl rollout status` / `describe`, and may run `kubectl rollout undo` manually.
> A simple improvement is to set `env.DEPLOYED = 'true'` *before* `rollout status`,
> or add an `unsuccessful` handler.

## Groovy / pipeline syntax cheat-sheet

| Syntax | Meaning |
|--------|---------|
| `'text ${X}'` | Literal — no Groovy interpolation (shell may expand `$X`) |
| `"text ${X}"` | Groovy interpolation |
| `'''multi\nline'''` | Multi-line literal |
| `env.NAME` | Pipeline-wide environment variable |
| `script { }` | Escape hatch to full Groovy inside declarative |
| `sh '...'` | Run shell command; non-zero exit = failure |
| `echo` | Print to the log |
| `withCredentials([...]) { }` | Scoped secret injection |

## Full timeline example (build #17, commit a1b2c3d)

1. Checkout → `IMAGE_TAG=17-a1b2c3d`
2. Test → builds `…/cicd-demo:test-17-a1b2c3d`, 4 tests pass
3. Build → `…/cicd-demo:17-a1b2c3d` and `…:latest`
4. Push → both tags uploaded to Docker Hub
5. Deploy → manifest rendered, applied; rolling update: 1 new pod created (surge), becomes Ready, 1 old pod removed, repeat
6. Smoke test → temp pod curls `/health` → `{"status":"ok"}`
7. Post → logout, remove test image. **SUCCESS**.

## Improvements to consider

* Replace `agent any` with Docker/Kubernetes agents.
* Add stages: lint, vulnerability scan (Trivy), integration tests.
* Archive artifacts / publish test reports (`junit`).
* Notifications (Slack/email) in `post`.
* Use `kubectl set image` or Kustomize instead of `sed`.
* Add manual approval (`input`) before production deploys (turns it into Continuous *Delivery*).
