# 12 — Kubernetes Fundamentals

## What Kubernetes (K8s) is

An open-source **container orchestrator**. You tell it the **desired state**
("run 2 copies of this image and expose them on a port"), and it continuously
works to make **actual state** match — restarting crashed containers, replacing
failed nodes, rolling out updates.

> **Declarative model + control loops.** You describe *what* you want in YAML;
> controllers constantly *reconcile* reality to match it.

## Cluster architecture

```
                    ┌────────────────── Control plane ───────────────────┐
 kubectl ──HTTPS──► │ kube-apiserver  ◄─► etcd (state database)          │
                    │ kube-scheduler (picks a node for each pod)         │
                    │ kube-controller-manager (Deployment, ReplicaSet…)  │
                    └───────────────────────────┬────────────────────────┘
                                                │
                    ┌──────────── Worker node(s) ▼ ─────────────────────┐
                    │ kubelet (runs pods, runs probes)                   │
                    │ kube-proxy (implements Service routing)            │
                    │ container runtime (containerd)                     │
                    │   [Pod] [Pod] [Pod]                                │
                    └────────────────────────────────────────────────────┘
```

In **kind** the control plane and the worker are the **same single node**
(role `control-plane`), itself a Docker container.

| Component | Job |
|-----------|-----|
| **kube-apiserver** | The front door; every `kubectl` call goes here |
| **etcd** | Key-value store of all cluster state |
| **scheduler** | Assigns unscheduled pods to nodes |
| **controller-manager** | Runs controllers (Deployment, ReplicaSet, Node…) |
| **kubelet** | Node agent; starts containers and executes probes |
| **kube-proxy** | Programs network rules so Services route to pods |

## Key objects used in this project

| Object | What it is |
|--------|-----------|
| **Namespace** | Virtual partition of a cluster (`cicd-demo`) |
| **Pod** | Smallest deployable unit: one or more containers sharing network/storage. Ephemeral |
| **ReplicaSet** | Keeps N identical pods running |
| **Deployment** | Manages ReplicaSets; adds rolling updates and rollback |
| **Service** | Stable virtual IP + DNS name that load-balances to matching pods |
| **HPA** *(optional)* | Autoscales replicas by CPU |
| **Ingress** *(optional)* | HTTP routing by host/path |

### How they relate

```
Deployment cicd-demo
   └─ ReplicaSet cicd-demo-<hash>   (one per template version)
        ├─ Pod cicd-demo-<hash>-aaaaa
        └─ Pod cicd-demo-<hash>-bbbbb
Service cicd-demo ──(label selector app=cicd-demo)──► the pods above
```

Why Deployments, not bare Pods: a bare pod that dies stays dead. A Deployment
replaces it. Pods get new names and IPs each time — which is why a **Service**
is needed.

## Labels and selectors

Labels are key/value tags (`app: cicd-demo`). **Selectors** find objects by label.
The Service sends traffic to pods whose labels match its selector; the Deployment
manages pods matching its `selector.matchLabels`. A mismatch = "Service has no endpoints".

## Services and types

| Type | Reachable from | Notes |
|------|---------------|-------|
| **ClusterIP** (default) | Inside the cluster only | Stable internal IP/DNS |
| **NodePort** *(used)* | Node IP + a port in 30000–32767 | Simple external access; here 30080 |
| LoadBalancer | Internet via cloud LB | Cloud providers |
| ExternalName | DNS alias | Rare |

Service DNS: `<service>.<namespace>.svc.cluster.local`; from the same namespace
just `cicd-demo` — exactly what the smoke test curls.

## Service port terminology (commonly confused)

```
port: 80          # the Service's own port (what clients inside the cluster use)
targetPort: 5000  # the container port traffic is forwarded to
nodePort: 30080   # the port opened on every node
```

## Probes and the pod lifecycle

* **Readiness probe** — "is this pod ready for traffic?" Failing ⇒ removed from Service endpoints (not restarted).
* **Liveness probe** — "is the process healthy?" Failing ⇒ kubelet **restarts** the container.
* **Startup probe** — protects slow-starting apps (not used).

Pod phases: `Pending → Running → Succeeded/Failed`. Common container states:
`ContainerCreating`, `Running`, `CrashLoopBackOff`, `ImagePullBackOff`, `Completed`.

## Requests and limits

* **Request**: amount the scheduler *reserves* for the pod.
* **Limit**: hard cap enforced at runtime. Exceeding memory limit ⇒ **OOMKilled**; exceeding CPU ⇒ throttled.
* Units: `50m` = 50 millicores = 0.05 CPU; `64Mi` = 64 mebibytes.

## Rolling updates and rollbacks

On changing the pod template (e.g., new image), the Deployment creates a **new
ReplicaSet** and scales it up while scaling the old one down, according to
`maxSurge`/`maxUnavailable`. Old ReplicaSets are retained
(`revisionHistoryLimit`) so you can:

```bash
kubectl rollout history deployment/cicd-demo -n cicd-demo
kubectl rollout undo    deployment/cicd-demo -n cicd-demo
kubectl rollout undo    deployment/cicd-demo -n cicd-demo --to-revision=3
```

## `kubectl` essentials

```bash
kubectl get pods,svc -n cicd-demo           # list resources
kubectl get pods -o wide                    # with node & IP
kubectl describe pod <name> -n cicd-demo    # events — first stop when debugging
kubectl logs -l app=cicd-demo --tail=50 -n cicd-demo
kubectl exec -it <pod> -n cicd-demo -- sh
kubectl apply -f file.yaml                  # create/update declaratively
kubectl delete -f file.yaml
kubectl rollout status deployment/cicd-demo -n cicd-demo
kubectl scale deployment/cicd-demo --replicas=3 -n cicd-demo
kubectl port-forward svc/cicd-demo 8081:80 -n cicd-demo
kubectl get events -n cicd-demo --sort-by=.lastTimestamp
kubectl config get-contexts
```

`apply` (declarative, idempotent) vs `create` (imperative, fails if it exists).

## Kubeconfig

A YAML file with three parts: **clusters** (API server address + CA), **users**
(credentials/certs) and **contexts** (cluster + user + namespace). `kubectl` finds
it via `$KUBECONFIG` or `~/.kube/config`. Treat it as a **secret** — it often grants admin access.
