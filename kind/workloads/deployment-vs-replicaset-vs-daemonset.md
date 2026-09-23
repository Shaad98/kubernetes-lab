
# Deployment vs ReplicaSet vs DaemonSet

Kubernetes provides **Deployment, ReplicaSet, and DaemonSet** to manage Pods, but they solve different problems.

---

## 1. Quick Difference

| Feature | Deployment | ReplicaSet | DaemonSet |
|---|---|---|---|
| Main purpose | Manage application releases | Maintain Pod count | Run a Pod on nodes |
| Creates Pods | Through ReplicaSet | Directly | Directly |
| `replicas` required | Yes | Yes | No |
| Rollout | Yes | No | Yes |
| Rollback | Yes | No built-in rollout rollback | Yes, for DaemonSet revisions |
| Image update handling | Creates new ReplicaSet and replaces Pods | Does **not** automatically replace existing Pods | Replaces Pods according to DaemonSet update strategy |
| Scaling | `kubectl scale` | `kubectl scale` | Not normally scaled with replicas |
| Pod on every node | No | No | Yes, according to node eligibility/selection |
| Best use case | Stateless applications | Low-level Pod replica management | Node-level agents |

---

# 2. Deployment

A Deployment is normally the **highest-level object** used to manage stateless applications.

```text
Deployment
    │
    ▼
ReplicaSet
    │
    ├── Pod
    ├── Pod
    └── Pod
````

For example:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx

  template:
    metadata:
      labels:
        app: nginx

    spec:
      containers:
        - name: nginx
          image: nginx:1.25
```

The Deployment creates and manages a ReplicaSet.

The ReplicaSet then maintains the required number of Pods.

---

## 2.1 Deployment Rollout

Suppose the current image is:

```yaml
image: nginx:1.25
```

You update it to:

```yaml
image: nginx:1.26
```

The Deployment creates a **new ReplicaSet**.

```text
Deployment
    │
    ├── Old ReplicaSet
    │       ├── Pod nginx:1.25
    │       ├── Pod nginx:1.25
    │       └── Pod nginx:1.25
    │
    └── New ReplicaSet
            ├── Pod nginx:1.26
            ├── Pod nginx:1.26
            └── Pod nginx:1.26
```

The Deployment gradually replaces the old Pods with new Pods according to its update strategy.

This is called a **rollout**.

You can see it with:

```bash
kubectl rollout status deployment/nginx
```

---

## 2.2 Deployment Rollback

Deployment keeps revision history.

For example:

```text
Revision 1 → nginx:1.25
Revision 2 → nginx:1.26
Revision 3 → nginx:1.27
```

If `nginx:1.27` causes a problem:

```bash
kubectl rollout undo deployment/nginx
```

Kubernetes can roll back to a previous revision.

You can check history:

```bash
kubectl rollout history deployment/nginx
```

So:

```text
Deployment
   │
   ├── rollout → move to new version
   │
   └── rollback → move back to previous version
```

---

# 3. ReplicaSet

A ReplicaSet has a much simpler responsibility:

> **Ensure that the desired number of Pods are running.**

Example:

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: nginx-rs

spec:
  replicas: 3

  selector:
    matchLabels:
      app: nginx

  template:
    metadata:
      labels:
        app: nginx

    spec:
      containers:
        - name: nginx
          image: nginx:1.25
```

The ReplicaSet ensures:

```text
Desired = 3

Pod 1
Pod 2
Pod 3
```

If one Pod dies:

```text
Before:

Pod 1
Pod 2
Pod 3

Pod 2 crashes

After:

Pod 1
Pod 3
New Pod
```

The ReplicaSet creates another Pod to maintain:

```text
replicas = 3
```

---

# 4. Important: ReplicaSet Does NOT Perform Rollouts

This is one of the most important differences.

Suppose your ReplicaSet currently has:

```yaml
image: nginx:1.25
```

You change it to:

```yaml
image: nginx:1.26
```

The existing Pods are **not automatically replaced** just because the ReplicaSet template changed.

For example:

```text
ReplicaSet

Template:
nginx:1.26

Existing Pods:

Pod 1 → nginx:1.25
Pod 2 → nginx:1.25
Pod 3 → nginx:1.25
```

The existing Pods continue running with:

```text
nginx:1.25
```

because Pods are already created objects.

---

## 4.1 What Happens When a Pod Is Deleted?

Suppose:

```text
ReplicaSet

Pod 1 → nginx:1.25
Pod 2 → nginx:1.25
Pod 3 → nginx:1.25
```

You delete Pod 2:

```bash
kubectl delete pod <pod-name>
```

ReplicaSet notices:

```text
Desired = 3
Current = 2
```

So it creates a replacement Pod using the **current ReplicaSet template**.

If you already changed the template to:

```yaml
image: nginx:1.26
```

the replacement Pod can be:

```text
Pod 1 → nginx:1.25
Pod 3 → nginx:1.25
Pod 4 → nginx:1.26
```

So you can temporarily have Pods running different image versions.

---

## 4.2 Scaling ReplicaSet

If you scale the ReplicaSet:

```bash
kubectl scale rs nginx-rs --replicas=5
```

The ReplicaSet needs two additional Pods.

Those new Pods are created from the **current ReplicaSet template**.

For example:

```text
Existing:

Pod 1 → nginx:1.25
Pod 2 → nginx:1.25
Pod 3 → nginx:1.25

Template was changed to:

nginx:1.26

Scale:

replicas = 5
```

New Pods:

```text
Pod 4 → nginx:1.26
Pod 5 → nginx:1.26
```

Result:

```text
Pod 1 → nginx:1.25
Pod 2 → nginx:1.25
Pod 3 → nginx:1.25
Pod 4 → nginx:1.26
Pod 5 → nginx:1.26
```

This is **not a rollout**.

ReplicaSet simply maintains the required number of Pods.

---

# 5. ReplicaSet vs Deployment

Think of it like this:

```text
ReplicaSet
    │
    └── "I only care that N Pods are running."

Deployment
    │
    ├── "I want N Pods running."
    ├── "I want controlled application updates."
    ├── "I want rollout status."
    └── "I want rollback."
```

Deployment uses ReplicaSets to provide these higher-level capabilities.

```text
Deployment
     │
     ├── ReplicaSet v1
     │      ├── Pod
     │      ├── Pod
     │      └── Pod
     │
     └── ReplicaSet v2
            ├── Pod
            ├── Pod
            └── Pod
```

---

# 6. DaemonSet

A DaemonSet solves a different problem.

Its purpose is:

> **Run one Pod on each eligible node.**

You normally do **not** specify:

```yaml
replicas: 3
```

Instead, Kubernetes determines how many Pods are needed based on the eligible nodes.

Example:

```text
Cluster

Node 1
Node 2
Node 3

DaemonSet

Node 1 → Pod
Node 2 → Pod
Node 3 → Pod
```

If a new eligible node joins:

```text
Before:

Node 1 → Pod
Node 2 → Pod
Node 3 → Pod

New Node 4 joins

After:

Node 1 → Pod
Node 2 → Pod
Node 3 → Pod
Node 4 → Pod
```

The DaemonSet automatically creates a Pod on Node 4.

---

# 7. DaemonSet Does Not Simply Mean "Every Node Including Control Plane"

An important correction:

A DaemonSet runs Pods on **every eligible node**, not necessarily every node.

For example, if your control-plane node has a taint that your DaemonSet does not tolerate:

```text
Control Plane
    │
    └── No DaemonSet Pod

Worker 1
    │
    └── DaemonSet Pod

Worker 2
    │
    └── DaemonSet Pod
```

So the common setup may look like:

```text
Control Plane → no Pod
Worker 1      → Pod
Worker 2      → Pod
Worker 3      → Pod
```

But this is because of **node eligibility/taints/tolerations**, not because DaemonSet has a hard-coded rule saying "never run on control plane."

---

# 8. DaemonSet Example

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: log-agent

spec:
  selector:
    matchLabels:
      app: log-agent

  template:
    metadata:
      labels:
        app: log-agent

    spec:
      containers:
        - name: log-agent
          image: fluentd:latest
```

Notice:

```yaml
replicas:
```

is not present.

Kubernetes decides how many Pods are needed based on the nodes where the DaemonSet should run.

---

# 9. DaemonSet Image Update

Suppose:

```yaml
image: log-agent:v1
```

is running on every eligible node.

You update it to:

```yaml
image: log-agent:v2
```

The DaemonSet can perform a controlled update of its Pods according to its update strategy.

For the default `RollingUpdate` strategy, Kubernetes replaces Pods progressively rather than simply leaving the old Pods forever.

You can check the rollout:

```bash
kubectl rollout status daemonset/log-agent
```

And view rollout history:

```bash
kubectl rollout history daemonset/log-agent
```

A DaemonSet also supports rollback:

```bash
kubectl rollout undo daemonset/log-agent
```

---

# 10. Deployment vs DaemonSet Image Update

This is a useful mental model.

## Deployment

```text
Deployment
    │
    ▼
New ReplicaSet
    │
    ▼
New Pods
```

The Deployment manages application versions through ReplicaSets.

---

## DaemonSet

```text
DaemonSet
    │
    ├── Node 1 → replace Pod
    ├── Node 2 → replace Pod
    └── Node 3 → replace Pod
```

The DaemonSet updates its node-level Pods according to its update strategy.

---

# 11. When Should You Use Each?

## Deployment

Use Deployment for normal stateless applications.

Examples:

```text
Spring Boot API
REST service
Frontend
Backend service
Web application
```

Typical configuration:

```yaml
replicas: 3
```

Think:

> "I need multiple instances of my application."

---

## ReplicaSet

Use ReplicaSet when you specifically need direct replica management.

However, in normal application deployments, you usually create a **Deployment instead of manually creating a ReplicaSet**.

Think:

> "I only need to maintain N identical Pods."

---

## DaemonSet

Use DaemonSet when you need a Pod associated with each eligible node.

Examples:

```text
Log collection agent
Node monitoring agent
Node-level security agent
Storage/networking components
```

Think:

> "I need this Pod on every eligible node."

---

# 12. Final Mental Model

The easiest way to remember everything:

```text
                 Kubernetes

        ┌────────────┼────────────┐
        │            │            │
        ▼            ▼            ▼
   Deployment    ReplicaSet    DaemonSet
        │            │            │
        │            │            │
        ▼            ▼            ▼
   Application    Pod count    Node-based
    releases       manager      Pods
        │
        └── ReplicaSet
              │
              ▼
             Pods
```

### Deployment

```text
"I manage application releases."

✓ replicas
✓ rollout
✓ rollback
✓ version history
✓ creates/manages ReplicaSets
✓ replaces Pods during updates
```

### ReplicaSet

```text
"I maintain the number of Pods."

✓ replicas
✓ replaces failed/deleted Pods
✗ no Deployment-style rollout
✗ no built-in Deployment revision workflow
✗ changing template does not automatically replace existing Pods
```

### DaemonSet

```text
"I maintain a Pod on every eligible node."

✓ no replicas field
✓ node-based
✓ automatically adds Pod when eligible node joins
✓ removes Pod when node is no longer eligible
✓ supports rolling updates
✓ supports rollback
```

---

# 13. One-Line Memory Trick

```text
Deployment → Manage APPLICATION RELEASES

ReplicaSet → Maintain POD COUNT

DaemonSet  → Maintain POD PER ELIGIBLE NODE
```

Or even simpler:

```text
Deployment = Version management
ReplicaSet = Number management
DaemonSet  = Node management
```

```

**One small but important correction to your original understanding:** a ReplicaSet doesn't wait specifically for a Pod to *crash* before using the new image. It only creates a replacement when it needs to create a Pod—for example, after a Pod is deleted or when you increase the replica count. Existing Pods are not proactively replaced just because you changed the ReplicaSet template.
```
