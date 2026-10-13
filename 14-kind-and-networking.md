# 14 — kind (Kubernetes IN Docker) and Networking

## What kind is

**kind** runs a real, conformant Kubernetes cluster where each **node is a Docker
container**. It was built for testing Kubernetes itself and is popular for CI and
local development. Alternatives: minikube, k3d, Docker Desktop Kubernetes.

```
Host (your laptop)
└── Docker engine
     ├── container "cicd-control-plane"   ← the kind node (runs kubelet, etcd, apiserver, containerd…)
     │     └── pods run inside it (as containerd containers)
     └── container "jenkins"
```

Naming: cluster `cicd` ⇒ node container `cicd-control-plane` (you'll see it in
`docker ps`, and in the README's `docker stop jenkins cicd-control-plane`).

## `kind/kind-config.yaml`

```yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
  - role: control-plane
    extraPortMappings:
      - containerPort: 30080
        hostPort: 8081
        protocol: TCP
```
* A **single-node** cluster (control-plane also runs workloads).
* `extraPortMappings` = Docker port publishing on the node container: **host 8081 → node container 30080**.
* Combined with the Service's `nodePort: 30080`, `http://localhost:8081` reaches your app.
* This must be set **at cluster creation** (it can't be changed later without recreating).
  That's why TROUBLESHOOTING says the cluster *must* be created with this file.
* Fallback without it: `kubectl -n cicd-demo port-forward svc/cicd-demo 8081:80`.

Why 8081 and not 8080? Port 8080 is Jenkins.

## Commands

```bash
kind create cluster --name cicd --config kind/kind-config.yaml
kind get clusters
kind get kubeconfig --name cicd              # external address (for your host)
kind get kubeconfig --internal --name cicd   # internal address (for containers on the kind network)
kind load docker-image cicd-demo:local --name cicd   # copy a local image into the node
kind delete cluster --name cicd
```

## Why `kind load` is needed

Cluster nodes use **their own** container runtime (containerd) with its own image
store — separate from your host Docker. Locally built images aren't visible
inside until you `kind load` them (or push to a registry).

## Networking walkthrough

### 1. Host → app
`localhost:8081` → Docker port map → node `:30080` → Service → pod `:5000`.

### 2. Jenkins → Kubernetes API
The API server listens on port `6443` inside the node container. Jenkins reaches
it using the **internal kubeconfig** *and* membership in the `kind` Docker network:

```
docker network connect kind jenkins
```
Both containers are then on the same virtual network and can resolve each other by name.

### 3. Pod → Service (smoke test)
The temporary curl pod resolves `cicd-demo` via **CoreDNS** to the Service's
ClusterIP, then kube-proxy forwards to a Ready pod.

### 4. Cluster → Docker Hub
Node pulls images over the internet (`ImagePullBackOff` if blocked/private/wrong name).

## Cluster DNS

CoreDNS runs in `kube-system`. Names:
`cicd-demo` (same namespace) → `cicd-demo.cicd-demo` → `cicd-demo.cicd-demo.svc.cluster.local`.

## Ports at a glance

| Port | Where | What |
|------|-------|------|
| 8080 | host → Jenkins | Web UI |
| 50000 | host → Jenkins | Agent connections |
| 8081 | host → kind node | App (maps to 30080) |
| 30080 | node | Service NodePort |
| 80 | Service | Cluster-internal service port |
| 5000 | pod/container | gunicorn |
| 6443 | node | Kubernetes API |

## Pause / resume behaviour

* `docker stop jenkins cicd-control-plane` frees CPU/RAM; cluster state persists on the container's filesystem.
* `docker start cicd-control-plane jenkins` brings it back (give the cluster a minute to become Ready).
* Jenkins state persists in the `jenkins_home` volume.
* A node IP can change after restart; the internal kubeconfig uses the container name for resilience.

## Windows-specific note from the README

The README generates a **UTF-8 (no BOM)** kubeconfig using .NET:
`[System.Text.UTF8Encoding]::new($false)`. PowerShell's default `>` redirection can
write UTF-16 or add a BOM, which `kubectl` cannot parse. Writing explicit UTF-8 avoids that.
