# 11 — Jenkins Container Setup and Configuration as Code (JCasC)

Folder: `jenkins/` — `Dockerfile`, `docker-compose.yml`, `docker-compose.casc.yml`,
`casc/jenkins.yaml`, `.env.example`.

## 1. `jenkins/Dockerfile` — a custom Jenkins image

The stock Jenkins image has neither the Docker CLI nor `kubectl`. This image adds them.

```dockerfile
FROM jenkins/jenkins:lts-jdk17          # Long-Term Support Jenkins on Java 17
USER root                               # need root to install packages
RUN apt-get update && apt-get install -y ca-certificates curl gnupg ...
```
**Docker CLI install** (only the client, not the daemon):
1. Create `/etc/apt/keyrings`, download Docker's GPG key into it.
2. Add Docker's apt repository (`signed-by=` pins that key — *supply-chain safety*).
3. `apt-get install docker-ce-cli docker-buildx-plugin`.
4. `rm -rf /var/lib/apt/lists/*` — shrinks the layer.

**kubectl install**:
```dockerfile
RUN ARCH=$(dpkg --print-architecture) \
 && curl --retry 3 --retry-delay 5 -fsSLo /usr/local/bin/kubectl \
    "https://dl.k8s.io/release/$(curl ... stable.txt)/bin/linux/${ARCH}/kubectl" \
 && chmod +x /usr/local/bin/kubectl
```
* Detects CPU architecture (`amd64` or `arm64`) so it also works on Apple Silicon.
* Fetches the latest stable version number, then that binary.
* `--retry` handles flaky networks.

```dockerfile
USER jenkins                            # drop back to unprivileged user
RUN jenkins-plugin-cli --plugins git workflow-aggregator credentials-binding \
    plain-credentials pipeline-stage-view timestamper configuration-as-code job-dsl
```
Pre-installs plugins so Jenkins starts fully equipped (no clicking in the UI).

## 2. `docker-compose.yml`

```yaml
services:
  jenkins:
    build: .
    container_name: jenkins
    user: root                      # demo only: lets Jenkins use the host Docker socket
    ports: ["8080:8080", "50000:50000"]
    volumes:
      - jenkins_home:/var/jenkins_home
      - /var/run/docker.sock:/var/run/docker.sock
volumes:
  jenkins_home:
```

| Key | Explanation |
|-----|-------------|
| `build: .` | Build the image from `jenkins/Dockerfile` |
| `container_name: jenkins` | Fixed name → used by `docker network connect kind jenkins` and `docker exec jenkins ...` |
| `user: root` | Runs the Jenkins process as root so it can use `/var/run/docker.sock` (which on Linux is owned by root:docker). Fixes the "permission denied …docker.sock" error, but is **not production-safe** |
| `8080:8080` | Web UI |
| `50000:50000` | Inbound agent (JNLP/WebSocket) port |
| `jenkins_home` volume | Persistent Jenkins state |
| Docker socket mount | Lets the Docker CLI inside Jenkins control the host's Docker daemon |

Docker Compose = define multi-container apps (and single ones) in YAML and
manage with `docker compose up/down`.

## 3. `docker-compose.casc.yml` — an *override* file

```yaml
services:
  jenkins:
    env_file: .env
    environment:
      - JAVA_OPTS=-Djenkins.install.runSetupWizard=false
      - CASC_JENKINS_CONFIG=/casc/jenkins.yaml
    volumes:
      - ./casc/jenkins.yaml:/casc/jenkins.yaml:ro
      - ../kubeconfig:/casc/kubeconfig:ro
```
Used as: `docker compose -f docker-compose.yml -f docker-compose.casc.yml up -d --build`.
Compose **merges** the files: the second adds to / overrides the first.

* `env_file: .env` — loads secrets from `jenkins/.env` into the container's environment.
* `runSetupWizard=false` — skip the "unlock Jenkins / install plugins" wizard.
* `CASC_JENKINS_CONFIG` — tells the JCasC plugin where the YAML lives.
* Mounts `jenkins.yaml` and the generated `kubeconfig` **read-only** (`:ro`).

This makes manual setup *optional*: without `.env`, `scripts/setup.sh` starts plain
Jenkins and prints the initial admin password for the wizard.

## 4. JCasC — `casc/jenkins.yaml`

**Configuration as Code** describes Jenkins' settings in YAML so a fresh Jenkins
configures itself identically every time (no snowflake servers, reviewable in Git).

### Section `jenkins:`
```yaml
systemMessage: "CI/CD demo Jenkins (configured by JCasC)"
numExecutors: 2
securityRealm:
  local:
    allowsSignup: false
    users:
      - id: "admin"
        password: "${JENKINS_ADMIN_PASSWORD:-admin123}"
authorizationStrategy:
  loggedInUsersCanDoAnything:
    allowAnonymousRead: false
```
* `${VAR:-default}` is **variable substitution with a default** (resolved from environment).
* The admin password comes from `.env`, defaulting to `admin123` (the README login).

### Section `credentials:`
```yaml
- usernamePassword: { scope: GLOBAL, id: "dockerhub-creds",
                      username: "${DOCKERHUB_USERNAME}", password: "${DOCKERHUB_TOKEN}" }
- file: { scope: GLOBAL, id: "kubeconfig", fileName: "kubeconfig",
          secretBytes: "${base64:${readFile:/casc/kubeconfig}}" }
```
* Creates both credentials the Jenkinsfile expects, with the **exact IDs**.
* `readFile:` reads the mounted kubeconfig; `base64:` encodes it as the plugin requires.
* This removes the error-prone manual UI steps.

### Section `jobs:` (Job DSL)
```groovy
pipelineJob('cicd-demo') {
  triggers { scm('H/2 * * * *') }
  definition { cpsScm { scm { git {
        remote { url('${GIT_REPO_URL}') }
        branch('*/${GIT_BRANCH:-main}') } }
      scriptPath('Jenkinsfile') } }
}
```
Creates the pipeline job automatically: use this Git URL, build the `main` branch, load `Jenkinsfile`, poll every ~2 minutes.

## 5. `.env.example` → `.env`

```
JENKINS_ADMIN_PASSWORD=change-me
DOCKERHUB_USERNAME=your-dockerhub-username
DOCKERHUB_TOKEN=your-dockerhub-access-token
GIT_REPO_URL=https://github.com/your-user/your-repo.git
GIT_BRANCH=main
```
`.env.example` is a **committed template**; the real `.env` is **git-ignored** because it contains secrets.

## 6. The kubeconfig: why `--internal`?

```
kind get kubeconfig --internal --name cicd > kubeconfig
```
Default kubeconfig points to `https://127.0.0.1:<random-port>` (reachable from your
host). But inside the Jenkins container, `127.0.0.1` is *Jenkins itself*, not the
cluster ("dial tcp 127.0.0.1" error). `--internal` writes the cluster's address as
seen **inside the Docker network** (`https://cicd-control-plane:6443`). Jenkins also
has to be attached to that network: `docker network connect kind jenkins`.

## 7. End-to-end bring-up order

1. `kind create cluster --name cicd --config kind/kind-config.yaml`
2. Generate `kubeconfig` (internal address) in the repo root
3. `docker compose -f jenkins/docker-compose.yml -f jenkins/docker-compose.casc.yml up -d --build`
4. `docker network connect kind jenkins`
5. Open http://localhost:8080 → job `cicd-demo` already exists → **Build Now**

## 8. Hardening for real life

| Demo shortcut | Production approach |
|---------------|---------------------|
| Jenkins as root + Docker socket | Rootless builders (Kaniko/Buildah), ephemeral agents |
| Static admin password in `.env` | SSO/LDAP/OIDC, secrets manager (Vault) |
| Kubeconfig with cluster-admin | A ServiceAccount with namespace-scoped RBAC |
| `loggedInUsersCanDoAnything` | Role-based authorization |
| Jenkins on `localhost` | TLS reverse proxy, backups of `jenkins_home` |
