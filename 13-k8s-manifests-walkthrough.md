# 13 — Kubernetes Manifests, Line by Line

Folder `k8s/`: `namespace.yaml`, `deployment.yaml`, `service.yaml`, plus
`optional/hpa.yaml` and `optional/ingress.yaml`.

## YAML refresher

* Indentation (spaces, never tabs) defines structure.
* `key: value` mapping; `- item` list entry; `{ a: b }` inline mapping.
* Every K8s object has: `apiVersion`, `kind`, `metadata`, and (mostly) `spec`.

## `namespace.yaml`

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: cicd-demo
```
A namespace scopes names, quotas and access. Everything else lives in `cicd-demo`,
keeping the demo separate from `default`/`kube-system`. Deleting the namespace deletes everything inside.

## `deployment.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cicd-demo
  namespace: cicd-demo
  labels: { app: cicd-demo }
spec:
  replicas: 2
  revisionHistoryLimit: 5
  strategy:
    type: RollingUpdate
    rollingUpdate: { maxSurge: 1, maxUnavailable: 0 }
  selector:
    matchLabels: { app: cicd-demo }
  template:
    metadata:
      labels: { app: cicd-demo }
    spec:
      containers:
        - name: cicd-demo
          image: DOCKER_IMAGE_PLACEHOLDER
          imagePullPolicy: IfNotPresent
          ports: [{ containerPort: 5000 }]
          readinessProbe: { httpGet: {path: /health, port: 5000}, initialDelaySeconds: 3,  periodSeconds: 5 }
          livenessProbe:  { httpGet: {path: /health, port: 5000}, initialDelaySeconds: 10, periodSeconds: 10 }
          resources:
            requests: { cpu: 50m,  memory: 64Mi }
            limits:   { cpu: 250m, memory: 128Mi }
```

### Field-by-field

| Field | Meaning / why |
|-------|---------------|
| `apiVersion: apps/v1` | API group for Deployments |
| `replicas: 2` | Two pods → high availability and visible load balancing |
| `revisionHistoryLimit: 5` | Keep 5 old ReplicaSets so `rollout undo` has something to return to |
| `strategy.type: RollingUpdate` | Gradual replacement (the default) |
| `maxSurge: 1` | May temporarily run **1 extra** pod above `replicas` (so up to 3) |
| `maxUnavailable: 0` | **Never** drop below 2 Ready pods → true zero downtime |
| `selector.matchLabels` | Which pods this Deployment owns; **must match** `template.metadata.labels` |
| `template` | The pod blueprint; changing it triggers a new rollout |
| `image: DOCKER_IMAGE_PLACEHOLDER` | Replaced by Jenkins' `sed` with `user/cicd-demo:BUILD-SHA`. The raw file isn't directly applicable |
| `imagePullPolicy: IfNotPresent` | Use cached image if the node has it (also enables `kind load`) |
| `containerPort: 5000` | Informational; documents the app port |

### Rolling update, step by step (replicas=2, surge=1, unavailable=0)

```
Start:      [old][old]
1. Surge:   [old][old][new]      new starts; NOT Ready yet, so no traffic
2. new Ready (readiness passes):  [old][old][new✔]
3. Remove an old:   [old][new✔]  (still 2 Ready pods — no dip)
4. Surge again:     [old][new✔][new]
5. new #2 Ready, remove last old: [new✔][new✔]   ← done
```
At no time are fewer than 2 Ready pods serving. If a new pod **never** becomes
Ready, the rollout stalls and old pods keep serving.

### Probes

```yaml
readinessProbe: httpGet /health:5000  initialDelaySeconds: 3   periodSeconds: 5
livenessProbe:  httpGet /health:5000  initialDelaySeconds: 10  periodSeconds: 10
```
* `httpGet`: kubelet sends GET; **200–399 = success**, else failure.
* `initialDelaySeconds`: wait before the first check (gives gunicorn time to boot).
* `periodSeconds`: check frequency.
* Defaults you aren't setting: `timeoutSeconds: 1`, `failureThreshold: 3`, `successThreshold: 1`.
* Liveness has a longer initial delay so a slow start isn't mistaken for a dead process.
* **Readiness** gates *traffic and rollout progress*; **liveness** triggers *restarts*.

### Resources

```yaml
requests: { cpu: 50m,  memory: 64Mi }    # reserved; used for scheduling
limits:   { cpu: 250m, memory: 128Mi }   # cap
```
* A pod with requests ≠ limits is in the **Burstable** QoS class (equal ⇒ Guaranteed; none ⇒ BestEffort).
* Why it matters: protects neighbours from a runaway pod and enables HPA (which computes utilization **relative to the CPU request**).
* Two gunicorn workers + Flask fit comfortably in 128Mi.

## `service.yaml`

```yaml
apiVersion: v1
kind: Service
metadata: { name: cicd-demo, namespace: cicd-demo }
spec:
  type: NodePort
  selector: { app: cicd-demo }
  ports:
    - port: 80
      targetPort: 5000
      nodePort: 30080
```

Traffic path:

```
localhost:8081 ─► (kind port mapping) ─► node:30080 ─► Service :80 ─► pod:5000
In-cluster client:  http://cicd-demo/health ─► Service :80 ─► pod:5000
```

* The `selector` finds pods by label `app=cicd-demo`; Kubernetes maintains an
  **Endpoints/EndpointSlice** list of Ready pod IPs.
* **Load balancing**: kube-proxy spreads connections across endpoints. That's why
  the README's loop `1..4 | % { (Invoke-RestMethod .../version).pod }` shows different pod names.
* `nodePort` must be in 30000–32767. Pinning it to `30080` is what makes the kind port mapping predictable.

## `optional/hpa.yaml` — Horizontal Pod Autoscaler

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
spec:
  scaleTargetRef: { apiVersion: apps/v1, kind: Deployment, name: cicd-demo }
  minReplicas: 2
  maxReplicas: 5
  metrics:
    - type: Resource
      resource: { name: cpu, target: { type: Utilization, averageUtilization: 60 } }
```
* Scales replicas between 2 and 5 to hold **average CPU ≈ 60% of the requested CPU** (60% × 50m = 30m per pod).
* Formula: `desired = ceil(current × currentUtilization / targetUtilization)`.
* Requires **metrics-server** (the file's header comment says to install it; on kind add `--kubelet-insecure-tls`).
* **Horizontal** = more pods. *Vertical* (VPA) = bigger pods.
* Note: don't also hard-code `replicas` in GitOps tools that fight the HPA.

## `optional/ingress.yaml`

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
spec:
  ingressClassName: nginx
  rules:
    - host: cicd-demo.local
      http:
        paths:
          - { path: /, pathType: Prefix, backend: { service: { name: cicd-demo, port: { number: 80 } } } }
```
* An **Ingress** is a set of HTTP(S) routing rules; an **Ingress controller**
  (ingress-nginx here) is the software that implements them. Without a controller the object does nothing.
* Routes `Host: cicd-demo.local` + any path → Service `cicd-demo:80`.
* You'd map `cicd-demo.local` to the node/LB IP in your `hosts` file.
* Benefits over NodePort: one entry point for many services, host/path routing, TLS termination.
* `pathType: Prefix` matches `/` and everything beneath it.

## `kubectl apply` order in the pipeline

1. `namespace.yaml` — must exist first
2. `deployment.rendered.yaml` — creates/updates ReplicaSet + pods
3. `service.yaml` — stable endpoint

Applying the same files repeatedly is safe: K8s computes a diff and changes only what differs.

## Verifying a deployment manually

```bash
kubectl -n cicd-demo get deploy,rs,pods,svc,endpoints
kubectl -n cicd-demo describe deployment cicd-demo
kubectl -n cicd-demo get deploy cicd-demo -o jsonpath='{.spec.template.spec.containers[0].image}'
kubectl -n cicd-demo rollout history deployment/cicd-demo
```

## Production-hardening ideas for the Deployment

* `securityContext`: `runAsNonRoot: true`, `readOnlyRootFilesystem: true`, `allowPrivilegeEscalation: false`, drop all capabilities.
* `topologySpreadConstraints` / anti-affinity so replicas land on different nodes.
* `PodDisruptionBudget` (e.g., `minAvailable: 1`) for safe node maintenance.
* NetworkPolicies, ConfigMaps/Secrets for configuration, `startupProbe`, `terminationGracePeriodSeconds`.
