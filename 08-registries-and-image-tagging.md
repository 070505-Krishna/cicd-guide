# 08 — Container Registries and Image Tagging

## What a registry is

A registry stores and distributes container images. Kubernetes nodes don't have
your laptop's images; they **pull** from a registry. In this project the registry
is **Docker Hub**.

Image name anatomy:

```
docker.io / likhithkumarp07 / cicd-demo : 12-ab12cd3
 registry     namespace/user     repo        tag
```
When the registry is omitted, Docker assumes `docker.io`.

## Authentication: Personal Access Token (PAT)

The README asks for a Docker Hub **PAT with Read, Write, Delete** permissions.

* A PAT replaces your password for automation; it can be scoped and revoked independently.
* Jenkins stores it as the credential `dockerhub-creds` (type *Username with password*).
* The pipeline logs in safely:

```bash
echo "$REG_PASS" | docker login -u "$REG_USER" --password-stdin
```
`--password-stdin` keeps the token out of the process list and shell history
(passing `-p TOKEN` would expose it).

Least privilege note: *Read & Write* is enough to push; *Delete* isn't required
by this pipeline.

## The tagging strategy used here

In the `Checkout` stage:

```groovy
env.GIT_SHORT = sh(returnStdout: true, script: 'git rev-parse --short HEAD').trim()
env.IMAGE_TAG = "${env.BUILD_NUMBER}-${env.GIT_SHORT}"
```

Example: Jenkins build **#17** on commit **`a1b2c3d`** → tag `17-a1b2c3d`.

Benefits:
* **Unique** per build → never overwritten.
* **Traceable** → tag tells you the build number *and* the exact commit.
* **Sortable** by build number.
* The running app shows this value at `/version` and on the home page.

The image is also tagged `:latest` (a moving pointer to the newest build).

### Why not rely on `:latest`?

| Problem with `latest` | Why it hurts |
|-----------------------|--------------|
| Mutable | The same tag points to different images over time |
| Untraceable | You can't tell what code is running |
| Breaks rollouts | If the manifest text doesn't change, Kubernetes sees "no change" and won't roll pods |
| Breaks rollback | `rollout undo` can't go back to "previous latest" |

That's why the deployment manifest gets the **unique tag**, not `latest`:
`sed "s|DOCKER_IMAGE_PLACEHOLDER|${IMAGE_NAME}:${IMAGE_TAG}|"`.
Every deploy changes the image string → Kubernetes creates a new ReplicaSet →
rolling update + clean rollback history.

## `imagePullPolicy`

The Deployment sets `imagePullPolicy: IfNotPresent`: the node pulls an image only
if it doesn't already have that exact tag. With unique tags this is safe and
fast. (`Always` re-checks the registry each time; K8s default is `Always` for
`:latest` and `IfNotPresent` otherwise.)

## Public vs private repositories

* **Public** Docker Hub repo → the cluster can pull without credentials (the simplest for the demo).
* **Private** repo → you must create an **imagePullSecret**:

```bash
kubectl -n cicd-demo create secret docker-registry regcred \
  --docker-username=USER --docker-password=PAT --docker-server=https://index.docker.io/v1/
```
and reference it in the pod spec: `imagePullSecrets: [{name: regcred}]`.

Hence `ImagePullBackOff` in the troubleshooting guide: the repo is private (or the username is wrong).

## Testing without any registry

`make deploy-local` builds locally and runs `kind load docker-image cicd-demo:local --name cicd`,
which copies the image **directly into the kind node**, bypassing Docker Hub.
(`imagePullPolicy: IfNotPresent` is what lets it use that local image.)

## Docker Hub considerations

* **Rate limits** on anonymous/free pulls exist; authenticated pulls get higher limits.
* Digests (`@sha256:...`) provide the strongest immutability guarantee.

## Other registries

| Registry | Typical use |
|----------|-------------|
| GitHub Container Registry (ghcr.io) | Next to GitHub repos |
| AWS ECR, Google Artifact Registry, Azure ACR | Cloud-native |
| Harbor | Self-hosted, with scanning/signing |
| Local `registry:2` | Offline demos |

## Image lifecycle best practices

* Retention policy: delete old tags automatically.
* Sign images (cosign) and verify at admission.
* Scan images on push and on schedule.
* Never put secrets in image layers (they're recoverable from history).
