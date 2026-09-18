# TaskHub — Production-Like Web Platform on Kubernetes (kind)

## 1. Project Overview

**TaskHub** is a small internal web platform where users view and manage tasks.
This repository does **not** build the TaskHub application itself — it builds
the Kubernetes infrastructure that runs, exposes, secures, and monitors it,
on a 3-node `kind` cluster (1 control-plane + 2 workers).

Kubernetes is responsible for:
- Scheduling and running the frontend, backend, and database containers
- Keeping the desired number of replicas alive (self-healing)
- Providing stable networking between components
- Persisting database data independently of Pod lifecycle
- Exposing the platform to users via Ingress
- Running one-off and scheduled maintenance tasks
- Enforcing least-privilege access for a node-level agent

This project demonstrates *where each Kubernetes object belongs and why*,
not a finished product.

## 2. Architecture Diagram

```
                         Users
                           |
                           v
                 Ingress Controller (nginx)
                           |
                        Ingress
                       /path   \path
                    /task       /api
                      |           |
                      v           v
              frontend-svc   backend-svc      (ClusterIP)
                (ClusterIP)   (ClusterIP)
                      |           |
                      v           v
               Frontend Pods  Backend Pods
               (Deployment,   (Deployment,
                3 replicas)    2 replicas)
                                   |
                                   v
                          database-headless (ClusterIP: None)
                                   |
                                   v
                          StatefulSet: database (1 replica)
                                   |
                                   v
                                  PVC  --->  PV  --->  hostPath storage

Also exposed directly (bypassing Ingress, for demonstration):
  frontend-nodeport   (NodePort, port 30080)
  frontend-loadbalancer (LoadBalancer)

Node-level, one per worker:
  DaemonSet: node-agent  (runs with ServiceAccount taskhub-agent)

Automation:
  Job: db-init-job          (single execution)
  Job: task-queue-job       (one completion, parallel workers)
  Job: batch-task-job       (multiple completions, parallel workers)
  CronJob: taskhub-cleanup  (scheduled every 5 minutes)

Security:
  ServiceAccount: taskhub-agent
        |
        v
  RoleBinding: taskhub-agent-binding
        |
        v
  Role: taskhub-pod-reader (get/list/watch Pods only)
```

## 3. Namespace and Labels

All TaskHub resources live in the `taskhub` Namespace, isolating this
project from anything else on the cluster.

Labeling strategy used throughout:

| Label        | Purpose                                      |
|--------------|-----------------------------------------------|
| `app: taskhub`         | Marks every resource that belongs to this project |
| `component: frontend`  | Frontend web layer |
| `component: backend`   | Backend/API layer |
| `component: database`  | Stateful database layer |
| `component: node-agent`| DaemonSet node agent |
| `component: job`       | One-off Jobs |
| `component: cronjob`   | Scheduled CronJob |
| `environment: lab`     | Marks this as a lab/training deployment |

This lets you answer, for example:
```
kubectl get pods -n taskhub -l app=taskhub          # all TaskHub pods
kubectl get pods -n taskhub -l component=backend    # only backend pods
kubectl get all -n taskhub -l component=database    # only database resources
```

## 4. Component Explanation

| Object | What it does | Why used here | Problem it solves |
|---|---|---|---|
| **Pod** | Smallest deployable unit; runs one or more containers | Every workload ultimately runs as Pods | Provides an isolated execution environment |
| **ReplicaSet** | Ensures N identical Pods are always running | Created automatically by each Deployment | Self-healing: replaces Pods that die |
| **Deployment** | Manages ReplicaSets and rolling updates for stateless apps | Frontend and backend | Declarative, safe way to release and scale stateless apps |
| **StatefulSet** | Manages Pods needing stable identity and storage | Database | Databases need a stable name and their own persistent volume, not interchangeable Pods |
| **DaemonSet** | Runs exactly one Pod per (eligible) node | `node-agent` | Node-level agents (log/metrics collectors) must exist on every node, not scaled by replica count |
| **Service (ClusterIP)** | Stable internal virtual IP + DNS name for a set of Pods | frontend-svc, backend-svc, database-headless | Frontend/backend/database must find each other reliably even as Pods restart |
| **Service (NodePort)** | Opens a static port on every node, forwarding to the Service | frontend-nodeport | Simple way to reach a Service from outside the cluster network |
| **Service (LoadBalancer)** | Requests an external load balancer from the cloud/infra provider | frontend-loadbalancer | Demonstrates the standard way production clusters expose Services externally |
| **Ingress + Ingress Controller** | HTTP(S) layer-7 router in front of Services | Routes `/task` → frontend, `/api` → backend | One external entry point instead of many NodePorts |
| **ConfigMap** | Stores non-sensitive configuration | `APP_NAME`, `APP_ENV`, `APP_PORT`, `LOG_LEVEL` | Decouples configuration from container images |
| **Secret** | Stores sensitive configuration | DB credentials, API token | Keeps sensitive values out of image builds and Git history |
| **PV / PVC** | Represents and requests persistent storage | Database data directory | Database data must outlive the Pod |
| **Job** | Runs a task to completion | DB init, queue worker, batch processing | One-off tasks that aren't long-running services |
| **CronJob** | Runs a Job on a schedule | `taskhub-cleanup` | Recurring maintenance without an external scheduler |
| **ServiceAccount** | Identity for a Pod talking to the Kubernetes API | `taskhub-agent` | node-agent needs an identity distinct from a human user |
| **Role** | Defines a set of permissions, scoped to a Namespace | `taskhub-pod-reader` | Grants only what's needed (get/list/watch Pods) |
| **RoleBinding** | Attaches a Role to a subject | `taskhub-agent-binding` | Connects the ServiceAccount's identity to its permissions |

### Deployment → ReplicaSet → Pods

```
Deployment ──manages──▶ ReplicaSet ──manages──▶ Pods
```

The **Deployment** is the object you actually edit (image version, replica
count, rollout strategy). It never touches Pods directly — it creates and
owns a **ReplicaSet**, which in turn creates and supervises the actual
**Pods**, constantly comparing "desired replicas" vs "current replicas" and
creating/deleting Pods to match.

**Why use a Deployment instead of creating a ReplicaSet directly?**
A bare ReplicaSet can keep N Pods alive, but it has no concept of a
*rollout*. A Deployment adds:
- Rolling updates (replace Pods gradually, not all at once)
- Rollback to a previous revision (`kubectl rollout undo`)
- Revision history
A ReplicaSet alone would force you to manually orchestrate every image
update by hand.

## 5. Storage Explanation

| Type | Used where in this project | Why |
|---|---|---|
| **emptyDir** | Frontend Pod, `/tmp/cache` | Temporary scratch space that only needs to live as long as the Pod. Wiped on Pod deletion — fine for a cache. |
| **hostPath** | Backing store for the database PV (`/mnt/data/taskhub-db` on the node) | Simplest way to get real persistence on a local `kind` lab cluster with no cloud storage provisioner available. In production this would be a cloud disk or network volume instead. |
| **NFS** | Not implemented in this lab | Would require running/mounting an actual NFS server, which is out of scope for a local `kind` cluster. Conceptually, NFS would be used to give **multiple** Pods simultaneous **ReadWriteMany** access to the same files (e.g. shared uploads directory) — something hostPath/PV here can't do since it's ReadWriteOnce. |
| **PV / PVC** | Database persistent data | The application (StatefulSet) should never depend on *how* storage is physically implemented. It asks for storage via a PVC; the PV is the actual piece of storage that satisfies that request. This decoupling means the same StatefulSet manifest works whether the underlying storage is hostPath (lab) or a cloud disk (production). |

Storage chain used for the database:
```
StatefulSet Pod → PVC (taskhub-db-pvc) → PV (taskhub-db-pv) → hostPath on node
```

> **kind-specific note:** hostPath here refers to a path *inside the kind
> node container*, not your physical machine. For the PV to reliably bind
> to data on disk across Pod recreation, either run a 1-node kind cluster or
> configure `extraMounts` in your kind config so the path is consistently
> backed by the host filesystem.

## 6. Networking Explanation

- **ClusterIP** — default, internal-only Service type. Used for
  `frontend-svc`, `backend-svc`, and the headless `database-headless`
  Service, since the backend and database should never be reachable
  directly from outside the cluster.
- **NodePort** — opens the same port (30080 here) on every node and forwards
  it to the Service. Simple way to test external access without an Ingress
  Controller, but doesn't scale well (one port per Service, no path-based
  routing).
- **LoadBalancer** — asks the platform for an external load balancer with
  its own IP. On `kind` there's no cloud controller, so `EXTERNAL-IP` stays
  `<pending>` unless you add MetalLB — but the object still demonstrates the
  correct API usage.
- **Ingress + Ingress Controller** — the actual HTTP entry point in this
  project. The Ingress Controller (nginx) watches Ingress objects and
  configures itself to route `/task` to `frontend-svc` and `/api` to
  `backend-svc`. This is the standard way to expose multiple HTTP services
  through a single external address with path-based routing.

## 7. Security Explanation (RBAC)

- **Subject**: the `taskhub-agent` ServiceAccount (used by the `node-agent`
  DaemonSet Pods as their identity when talking to the API server).
- **Role**: `taskhub-pod-reader`, scoped to the `taskhub` Namespace, granting
  only `get`, `list`, `watch` on `pods`.
- **RoleBinding**: `taskhub-agent-binding` connects the ServiceAccount to the
  Role.
- **Permissions granted**: the node-agent can inspect Pods in the `taskhub`
  Namespace only — it cannot modify Pods, cannot touch Secrets/ConfigMaps,
  and cannot see resources in other Namespaces.
- **Why these permissions**: a node-level monitoring agent's realistic job
  is to observe what's running (e.g. to report Pod health), not to change
  anything. Granting only read verbs on one resource type is the **least
  privilege** needed for that job.

Verify with:
```
kubectl auth can-i list pods --as=system:serviceaccount:taskhub:taskhub-agent -n taskhub      # yes
kubectl auth can-i delete pods --as=system:serviceaccount:taskhub:taskhub-agent -n taskhub    # no
kubectl auth can-i list secrets --as=system:serviceaccount:taskhub:taskhub-agent -n taskhub   # no
```

## 8. Jobs and CronJobs

| Job | Pattern | completions | parallelism | Behavior |
|---|---|---|---|---|
| `db-init-job` | Single execution | 1 (default) | 1 (default) | Runs once, exits, done. |
| `task-queue-job` | One completion, parallel executions ("work queue") | *(omitted)* | 3 | Up to 3 Pods run at once; the Job is marked complete as soon as **any one** exits successfully. |
| `batch-task-job` | Multiple completions, parallel executions | 5 | 2 | Needs 5 total successful Pod completions; never runs more than 2 Pods at the same time. |

`completions` sets *how many total successes are required*.
`parallelism` caps *how many Pods may run concurrently* while working
towards that total.

**CronJob flow:**
```
CronJob (schedule: */5 * * * *) → creates a Job on each tick → Job creates a Pod → Pod runs the task and exits
```
`taskhub-cleanup` keeps the last 3 successful and 1 failed Job history for
inspection (`kubectl get jobs -n taskhub`).

## 9. Health and Resources

- **Startup Probe** — "has the app finished starting?" Runs first; while it
  is failing, liveness/readiness probes are disabled so a slow-starting
  container isn't killed prematurely.
- **Readiness Probe** — "can the app receive traffic right now?" If it
  fails, the Pod is removed from Service endpoints (no traffic sent) but is
  **not** restarted.
- **Liveness Probe** — "is the app still healthy?" If it fails repeatedly,
  Kubernetes **restarts** the container.

All main containers (frontend, backend, database) define:
- **CPU/Memory Requests** — the guaranteed minimum the scheduler reserves;
  used to decide which node a Pod fits on.
- **CPU/Memory Limits** — the hard ceiling; the container is throttled (CPU)
  or OOM-killed (memory) if it tries to exceed this.

Frontend/backend use small values (`50m`/`64Mi` request, `200m`/`128Mi`
limit) since they're lightweight demo containers; the database gets more
(`100m`/`128Mi` request, `300m`/`256Mi` limit) since Postgres needs more
headroom even at idle.

## 10. Failure and Recovery Tests

### Test 1 — Break a Health Check
1. **Problem**: edit the frontend Deployment's readiness probe path to a
   non-existent path, e.g. `/does-not-exist`, and `kubectl apply` it.
2. **Investigation**: `kubectl get pods -n taskhub` — Pod shows `READY 0/1`.
   `kubectl describe pod <pod> -n taskhub` — Events show failed probe.
3. **Root Cause**: probe is checking a path the container doesn't serve.
4. **Fix**: revert the probe path to `/`.
5. **Recovery**: `kubectl get pods -n taskhub -w` — Pod returns to
   `READY 1/1` once the probe starts succeeding again.

### Test 2 — Delete an Application Pod
```
kubectl delete pod <frontend-pod-name> -n taskhub
kubectl get pods -n taskhub -w
```
A replacement Pod appears almost immediately.
**Why**: the ReplicaSet (owned by the Deployment) continuously compares
desired vs. actual replica count and creates a new Pod the moment it sees
one missing.
**Responsible object**: the ReplicaSet.

## 11. Stateful Storage Test

```
kubectl exec -it database-0 -n taskhub -- psql -U taskhub_user -d taskhub \
  -c "CREATE TABLE demo(id serial primary key, note text); INSERT INTO demo(note) VALUES ('hello');"

kubectl delete pod database-0 -n taskhub
kubectl wait --for=condition=Ready pod/database-0 -n taskhub --timeout=120s

kubectl exec -it database-0 -n taskhub -- psql -U taskhub_user -d taskhub \
  -c "SELECT * FROM demo;"
```
The row inserted before deletion is still there after the Pod is recreated,
because the data lives on the PVC/PV — not inside the container filesystem.

**Key concept**: *Pod lifecycle ≠ Data lifecycle.* Deleting/recreating the
Pod gives you a brand-new container, but it reattaches to the **same**
PersistentVolumeClaim, so the data survives independently of the Pod.

## 12. Deployment Order

```
kubectl apply -f namespace/
kubectl apply -f config/
kubectl apply -f frontend/
kubectl apply -f backend/
kubectl apply -f database/pv.yaml -f database/pvc.yaml
kubectl apply -f database/statefulset.yaml -f database/service.yaml
kubectl apply -f networking/nodeport.yaml -f networking/loadbalancer.yaml
kubectl apply -f security/
kubectl apply -f daemonset/
kubectl apply -f networking/ingress.yaml   # after an Ingress Controller is installed
kubectl apply -f jobs/
kubectl apply -f cronjob/
```

## 13. Repository Structure

```
k8s-taskhub/
├── namespace/namespace.yaml
├── config/configmap.yaml, secret.yaml
├── frontend/deployment.yaml, service.yaml
├── backend/deployment.yaml, service.yaml
├── database/statefulset.yaml, service.yaml, pv.yaml, pvc.yaml
├── networking/nodeport.yaml, loadbalancer.yaml, ingress.yaml
├── daemonset/daemonset.yaml
├── jobs/single-job.yaml, parallel-job.yaml, multi-completion-job.yaml
├── cronjob/cronjob.yaml
├── security/serviceaccount.yaml, role.yaml, rolebinding.yaml
├── screenshots/        (add your kubectl get/describe screenshots here)
└── README.md
```

## 14. Evidence Checklist (fill in with screenshots from your cluster)

- [ ] `kubectl get nodes -o wide`
- [ ] `kubectl get ns taskhub`
- [ ] `kubectl get pods -n taskhub -l component=frontend`
- [ ] `kubectl get pods -n taskhub -l component=backend`
- [ ] `kubectl get statefulset,pods -n taskhub -l component=database`
- [ ] `kubectl get rs -n taskhub`
- [ ] `kubectl get daemonset -n taskhub -o wide`
- [ ] `kubectl get svc -n taskhub`
- [ ] `kubectl get ingress -n taskhub`
- [ ] `kubectl get pv,pvc -n taskhub`
- [ ] `kubectl get configmap,secret -n taskhub`
- [ ] `kubectl get jobs -n taskhub`
- [ ] `kubectl get cronjob -n taskhub`
- [ ] `kubectl get sa,role,rolebinding -n taskhub`
- [ ] `kubectl describe pod <frontend-pod> -n taskhub` (resources + probes section)
