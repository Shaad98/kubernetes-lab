# Kubernetes — Complete Beginner-to-Intermediate Notes

> A practical introduction to Kubernetes: why it exists, how the architecture works, what each major component does, how `kubectl` fits in, how networking works with CNI, and how the most important Kubernetes objects are used.

---

# 1. What Is Kubernetes?

Kubernetes, commonly called **K8s**, is an open-source platform for **deploying, running, scaling, networking, and managing containerized applications**.

Suppose you have a Spring Boot application.

Without Kubernetes, you might manually:

1. Start containers.
2. Decide which server should run each container.
3. Restart containers after crashes.
4. Increase the number of containers when traffic increases.
5. Connect containers to each other.
6. Expose an application to users.
7. Roll out a new application version.
8. Roll back a bad deployment.
9. Keep track of which containers are healthy.
10. Handle machines that fail.

Kubernetes automates these responsibilities.

A useful mental model is:

```text
You declare:

"I want 3 copies of my application running."

                 ↓

            Kubernetes
                 ↓

   Continuously works to make
   reality match your request.
```

This is the core idea of Kubernetes:

> **Desired state → Kubernetes controllers → Actual state matches desired state**

---

# 2. Why Do We Need Kubernetes?

## 2.1 Containerization alone is not enough

Docker solves an important problem:

```text
Application + Dependencies
            ↓
         Container
```

But when you have many containers and many machines, new problems appear.

For example:

```text
                    Users
                      |
                 Load Balancer
                      |
        +-------------+-------------+
        |             |             |
      Server 1      Server 2      Server 3
        |             |             |
      App A         App A         App A
      App B         App B         App B
```

Now imagine one application crashes.

Who notices?

Who restarts it?

What if Server 2 fails?

Who moves workloads somewhere else?

What if traffic suddenly becomes 10× larger?

Who increases the number of application instances?

What if you release version 2 and it is broken?

Who rolls back to version 1?

These are the problems Kubernetes solves.

---

# 3. What Kubernetes Gives You

Kubernetes provides mechanisms for:

### Scheduling

Determines **which node should run a Pod**.

### Self-healing

Kubernetes can recreate workloads when they fail.

### Scaling

You can increase or decrease the number of application replicas.

### Service discovery

Applications can discover other applications using stable Kubernetes Services.

### Load balancing

Traffic can be distributed across healthy Pods.

### Rolling updates

New application versions can be introduced gradually.

### Rollbacks

A failed release can be rolled back.

### Configuration management

Configuration and sensitive data can be managed separately from container images.

### Storage orchestration

Persistent storage can be attached to workloads.

### Networking

Pods can communicate across nodes through a cluster networking implementation.

---

# 4. Kubernetes Architecture

Kubernetes follows a **Control Plane + Worker Node** architecture.

```text
                    Kubernetes Cluster
                           |
             +-------------+-------------+
             |                           |
       CONTROL PLANE                 WORKER NODES
       manages cluster               run workloads
             |                           |
     +-------+-------+          +--------+--------+
     |       |       |          |        |        |
 API Server etcd  Scheduler    kubelet  runtime  kube-proxy
     |       |       |                     |
     +-------+-------+                     |
             |                             Pods
       Controller Manager
```

The easiest way to think about it is:

> **Control Plane decides and coordinates. Worker Nodes execute workloads.**

---

# 5. Control Plane

The main Control Plane components are:

1. `kube-apiserver`
2. `etcd`
3. `kube-scheduler`
4. `kube-controller-manager`

In cloud environments you may also have:

5. `cloud-controller-manager`

The first four are the standard core Control Plane components.

---

# 6. kube-apiserver

The **kube-apiserver** is the central API entry point of Kubernetes.

Nearly every Kubernetes operation begins by interacting with the API server.

For example:

```text
kubectl apply -f deployment.yaml
              |
              v
       kube-apiserver
              |
       validates request
              |
        stores state
              |
             etcd
```

## Responsibilities

- Exposes the Kubernetes REST API.
- Authenticates requests.
- Authorizes requests.
- Validates Kubernetes objects.
- Acts as the central interface for cluster state.
- Allows other Kubernetes components to interact with the cluster.

## Important idea

The API server is **not the component that directly starts your containers**.

For example:

```text
API Server
    |
    | Pod specification
    v
kubelet
    |
    v
Container Runtime
    |
    v
Container
```

The API server coordinates the Kubernetes API; the actual workload execution happens on worker nodes.

---

# 7. etcd

`etcd` is the distributed key-value database used by Kubernetes.

You can think of it as:

> **Kubernetes' source of truth for cluster state.**

It stores Kubernetes state such as:

- Pods
- Deployments
- ReplicaSets
- Services
- ConfigMaps
- Secrets
- Nodes
- Namespaces
- Metadata
- Desired configuration

A simplified example:

```text
Deployment
replicas = 3
image = my-app:v1
```

That desired configuration is persisted as Kubernetes state.

## Important

`etcd` does **not** run containers.

It stores state.

Think:

```text
etcd       = database
kubelet    = node agent
runtime    = container executor
```

---

# 8. kube-scheduler

The `kube-scheduler` decides **which worker node should run an unscheduled Pod**.

Suppose you create:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app
spec:
  containers:
    - name: app
      image: my-app:v1
```

The Pod exists in the Kubernetes API, but it does not yet have a selected node.

The scheduler evaluates available nodes.

It can consider:

- CPU and memory requirements
- Node availability
- Resource requests
- Taints and tolerations
- Node affinity
- Pod affinity / anti-affinity
- Scheduling policies

Then it makes a decision:

```text
Pod
 |
 | scheduler chooses
 v
Worker Node 2
```

## Important

The scheduler does **not** normally start the container.

Its main responsibility is:

> **Choose where the Pod should run.**

---

# 9. kube-controller-manager

The `kube-controller-manager` runs multiple controllers.

Controllers continuously compare:

```text
Desired State
      |
      v
Actual State
```

If they differ, a controller takes action.

Example:

```text
Desired:
3 replicas

Actual:
2 Pods

      ↓

Controller detects difference

      ↓

Create another Pod

      ↓

Actual:
3 Pods
```

Examples of controllers include:

- Deployment controller
- ReplicaSet controller
- Node controller
- Job controller
- Namespace controller
- ServiceAccount controller

This is one of the most important Kubernetes ideas:

> **Controllers continuously work to make actual state match desired state.**

---

# 10. cloud-controller-manager

The `cloud-controller-manager` is used when Kubernetes needs to interact with a cloud provider.

It is typically found in cloud environments such as AWS, Azure, or Google Cloud.

It can integrate Kubernetes with cloud resources such as:

- Load balancers
- Routes
- Cloud volumes
- Cloud-specific node information

Conceptually:

```text
Kubernetes
    |
    v
cloud-controller-manager
    |
    v
Cloud Provider APIs
    |
    +---- AWS
    +---- Azure
    +---- GCP
```

It is **cloud-specific and optional**, unlike the basic Kubernetes components.

---

# 11. Worker Nodes

Worker Nodes are the machines where application workloads run.

A worker node commonly contains:

```text
Worker Node
│
├── kubelet
├── Container Runtime
├── kube-proxy (in many/common setups)
└── Pods
```

The three important node-side components in this architecture are:

1. `kubelet`
2. Container Runtime
3. `kube-proxy`

CNI is the networking layer that provides Pod networking.

---

# 12. kubelet

The `kubelet` is the main agent running on each worker node.

It communicates with the Kubernetes API and makes sure the Pods assigned to its node are actually running as expected.

Simplified flow:

```text
kube-apiserver
       |
       | Pod specification
       v
     kubelet
       |
       v
Container Runtime
       |
       v
   Container
```

## Responsibilities

- Watches for Pod specifications assigned to the node.
- Ensures required containers are running.
- Reports Pod status.
- Reports node status.
- Executes health-related lifecycle actions.
- Works with the container runtime.

## Important

`kubelet` is **not the container runtime**.

It tells the runtime what should run.

---

# 13. Container Runtime

The container runtime is responsible for actually managing containers.

Common Kubernetes-compatible runtimes include:

- `containerd`
- `CRI-O`

The runtime can:

- Pull container images.
- Create containers.
- Start containers.
- Stop containers.
- Remove containers.
- Manage container lifecycle.

The simplified relationship is:

```text
kubelet
   |
   v
Container Runtime
   |
   +---- pulls image
   +---- creates container
   +---- starts container
   |
   v
Application Container
```

## What about Docker?

Historically Kubernetes commonly used Docker Engine.

Modern Kubernetes uses the **Container Runtime Interface (CRI)** and common runtimes include containerd and CRI-O.

Docker can still be part of a development workflow, but it should not be mentally confused with the Kubernetes control-plane components.

---

# 14. kube-proxy

`kube-proxy` is traditionally used on worker nodes to implement Kubernetes Service networking rules.

It helps traffic reach the Pods behind Services.

Conceptually:

```text
Client
  |
  v
Service
  |
  v
Service networking rules
  |
  +---- Pod 1
  +---- Pod 2
  +---- Pod 3
```

Depending on the cluster and networking implementation, node traffic rules may use mechanisms such as:

- iptables
- IPVS

Some modern Kubernetes networking solutions use eBPF-based implementations and may reduce or replace the traditional kube-proxy role.

So:

> **Do not think of kube-proxy as the component that runs Pods.**

It is primarily associated with **Service networking on nodes**.

---

# 15. CNI — Container Network Interface

CNI is one of the most important concepts in Kubernetes networking.

Kubernetes needs a networking implementation that provides connectivity for Pods.

A CNI implementation can provide:

- Pod IP assignment
- Pod-to-Pod connectivity
- Cross-node Pod networking
- Network configuration
- Additional network policies and networking features depending on the plugin

Examples include:

- Calico
- Cilium
- Flannel
- Weave Net

A simplified model:

```text
Pod A
  |
  +---------+
            |
           CNI
            |
  +---------+
  |
Pod B
```

For cross-node communication:

```text
Node 1                         Node 2

Pod A                          Pod B
  |                              |
  +----------- CNI --------------+
```

## CNI is not a core Control Plane process

This distinction is important.

```text
Core Control Plane:
- kube-apiserver
- etcd
- scheduler
- controller-manager

Node components:
- kubelet
- container runtime
- kube-proxy

Networking:
- CNI plugin
```

---

# 16. kubectl

`kubectl` is the command-line client used to communicate with a Kubernetes cluster.

It is **not a Kubernetes Control Plane component**.

Think of it as:

```text
Your terminal
     |
     v
   kubectl
     |
     v
kube-apiserver
     |
     +---- etcd
     +---- scheduler
     +---- controllers
     +---- worker nodes
```

Examples:

```bash
kubectl get pods
kubectl get nodes
kubectl get services
kubectl get deployments
kubectl describe pod my-pod
kubectl logs my-pod
kubectl exec -it my-pod -- /bin/sh
kubectl apply -f deployment.yaml
kubectl delete -f deployment.yaml
```

---

# 17. What Happens When You Run kubectl apply?

Consider:

```bash
kubectl apply -f deployment.yaml
```

A useful simplified flow is:

```text
1. kubectl
      |
      v
2. kube-apiserver
      |
      v
3. Validation + admission processing
      |
      v
4. State stored in etcd
      |
      v
5. Controllers observe desired state
      |
      v
6. Deployment creates/updates ReplicaSet
      |
      v
7. ReplicaSet creates Pods
      |
      v
8. Scheduler selects nodes for unscheduled Pods
      |
      v
9. kubelet sees assigned Pods
      |
      v
10. Container runtime starts containers
      |
      v
11. CNI configures Pod networking
      |
      v
12. Pod runs
```

This is a **simplified educational flow**.

In real Kubernetes, multiple watches, reconciliation loops, API operations, status updates, and networking steps happen concurrently.

---

# 18. Important Communication Model

Do not memorize Kubernetes as one giant linear pipeline.

A better mental model is:

```text
                       kube-apiserver
                      /      |       \
                     /       |        \
                  etcd   scheduler   controllers
                                      |
                                      |
                                   API state
                                      |
                            +---------+---------+
                            |                   |
                          Node 1              Node 2
                            |                   |
                         kubelet              kubelet
                            |                   |
                          runtime             runtime
                            |                   |
                           Pods                Pods
```

The Kubernetes API server is central to Kubernetes component communication.

However:

> **Pod application traffic does not normally travel through the kube-apiserver.**

Pod networking is provided by the cluster network/CNI and Services.

---

# 19. Kubernetes Objects

Kubernetes stores and manages resources as **objects**.

Objects are declarations of the state you want Kubernetes to maintain.

Examples:

- Pod
- Deployment
- ReplicaSet
- Service
- DaemonSet
- Job
- Namespace
- StatefulSet
- ConfigMap
- Secret
- PersistentVolume
- PersistentVolumeClaim
- Ingress

Now let's focus on the most important ones.

---

# 20. Pod

A **Pod** is the smallest deployable unit in Kubernetes.

A Pod usually contains one application container.

Example:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  containers:
    - name: nginx
      image: nginx
      ports:
        - containerPort: 80
```

Run:

```bash
kubectl apply -f pod.yaml
```

## Pod can contain multiple containers

Example:

```text
Pod
├── main application container
└── sidecar container
```

Containers in the same Pod share:

- Network namespace
- Pod IP
- Local networking
- Can share volumes

## Important

A Pod is **ephemeral**.

Do not think:

> "This exact Pod is my permanent server."

Instead think:

> "Kubernetes can replace Pods."

---

# 21. Why We Usually Don't Create Pods Directly

You can create:

```yaml
kind: Pod
```

But production workloads are usually managed through higher-level controllers.

For example:

```text
Deployment
    |
    v
ReplicaSet
    |
    v
Pods
```

Why?

Because controllers provide:

- Replication
- Self-healing
- Rolling updates
- Rollbacks
- Declarative management

---

# 22. ReplicaSet

A **ReplicaSet** ensures that a specified number of matching Pods are running.

Example:

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: app-rs
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
        - name: app
          image: my-app:v1
```

If:

```text
Desired = 3 Pods
Actual  = 2 Pods
```

ReplicaSet works toward:

```text
Actual = 3 Pods
```

## Why ReplicaSet?

Replication and self-healing at the Pod level.

But usually you should not manage ReplicaSets directly.

A Deployment normally manages them.

---

# 23. Deployment

A **Deployment** is the common way to manage stateless applications.

Typical relationship:

```text
Deployment
    |
    v
ReplicaSet
    |
    v
Pods
```

Example:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
        - name: app
          image: my-app:v1
          ports:
            - containerPort: 8080
```

## Deployment gives you

- Replica management
- Rolling updates
- Rollbacks
- Declarative updates
- Self-healing through ReplicaSets
- Scaling

Scale:

```bash
kubectl scale deployment my-app --replicas=5
```

---

# 24. Deployment vs ReplicaSet vs Pod

```text
Deployment
    |
    | manages
    v
ReplicaSet
    |
    | manages
    v
Pods
```

### Pod

Runs the actual application containers.

### ReplicaSet

Maintains the requested number of matching Pods.

### Deployment

Manages ReplicaSets and provides application rollout/rollback behavior.

A practical rule:

> **Create Deployments for most stateless applications instead of creating Pods directly.**

---

# 25. Service

A Kubernetes **Service** provides a stable network endpoint for a group of Pods.

Pods are replaceable and their IP addresses can change.

Suppose:

```text
Pod A = 10.1.0.5
Pod B = 10.1.0.6
Pod C = 10.1.0.7
```

A client should not hard-code these Pod IPs.

Instead:

```text
Client
   |
   v
Service
   |
   +---- Pod A
   +---- Pod B
   +---- Pod C
```

The Service gives stable discovery and routing.

---

# 26. Common Service Types

## ClusterIP

Default type.

Accessible inside the cluster.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: backend
spec:
  selector:
    app: backend
  ports:
    - port: 80
      targetPort: 8080
```

Used for internal application-to-application communication.

Example:

```text
frontend
   |
   v
backend Service
   |
   +---- backend Pod
   +---- backend Pod
```

---

## NodePort

Exposes a Service through a port on each node.

```text
Client
  |
  v
NodeIP:NodePort
  |
  v
Service
  |
  v
Pod
```

Useful for learning and simple access, but usually not the preferred production exposure mechanism by itself.

---

## LoadBalancer

Requests an external load balancer through the cloud/provider integration.

Typical model:

```text
Internet
   |
   v
Cloud Load Balancer
   |
   v
Service
   |
   +---- Pod
   +---- Pod
```

---

# 27. Deployment + Service

A very common application architecture is:

```text
                    Internet
                       |
                       v
                    Service
                       |
          +------------+------------+
          |            |            |
         Pod          Pod          Pod
          ^            ^            ^
          |            |            |
          +-------- Deployment -----+
                       |
                   ReplicaSet
```

The Service gives a stable endpoint.

The Deployment manages the Pods.

---

# 28. DaemonSet

A **DaemonSet** ensures that a Pod runs on every eligible node.

Example:

```text
Node 1 → monitoring Pod
Node 2 → monitoring Pod
Node 3 → monitoring Pod
```

When a new node joins:

```text
Node 4 joins
      |
      v
DaemonSet creates Pod on Node 4
```

Typical uses:

- Log collectors
- Monitoring agents
- Node-level security agents
- CNI-related components
- Storage/network node agents

## Key idea

Deployment asks:

> "How many replicas do I want?"

DaemonSet asks:

> "Which nodes should have one copy of this Pod?"

---

# 29. Deployment vs DaemonSet

| Feature | Deployment | DaemonSet |
|---|---|---|
| Main purpose | Run application replicas | Run one Pod per eligible node |
| Replica model | 1, 2, 3, 10... | Usually one per eligible node |
| Typical use | Web/API applications | Logging/monitoring/node agents |
| New node joins | Not automatically one Pod per node | DaemonSet creates a Pod there |
| Node-focused | No | Yes |

Example:

```text
Deployment:

Node 1: Pod
Node 2: Pod
Node 3: Pod
Node 1: Pod

Total = 4 replicas


DaemonSet:

Node 1: Pod
Node 2: Pod
Node 3: Pod

Total = one per eligible node
```

---

# 30. Job

A **Job** is used for a task that should run to completion.

For example:

```text
Run database migration once
Process a batch
Generate a report
Run a cleanup script
```

Example:

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: hello-job
spec:
  template:
    spec:
      restartPolicy: Never
      containers:
        - name: task
          image: busybox
          command: ["sh", "-c", "echo Hello Kubernetes"]
```

A Job is different from a Deployment.

### Deployment

```text
Keep application running
```

### Job

```text
Run task until completion
```

---

# 31. Job vs Deployment

| Feature | Deployment | Job |
|---|---|---|
| Purpose | Long-running application | Finite task |
| Completion | Keeps running | Ends successfully |
| Example | REST API | Database migration |
| Restart model | Maintains replicas | Retries failed jobs according to policy |
| Controller | Deployment/ReplicaSet chain | Job controller |

---

# 32. Namespace

A **Namespace** provides a logical boundary inside a Kubernetes cluster.

Suppose one cluster contains:

```text
development
staging
production
```

You can organize resources:

```text
Cluster
|
+-- namespace: dev
|     +-- deployments
|     +-- services
|     +-- pods
|
+-- namespace: staging
|     +-- deployments
|     +-- services
|     +-- pods
|
+-- namespace: production
      +-- deployments
      +-- services
      +-- pods
```

Create a namespace:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: dev
```

Then deploy into it:

```bash
kubectl apply -f deployment.yaml -n dev
```

## Why use Namespaces?

They help with:

- Organization
- Resource isolation
- Team separation
- Access control
- Resource quotas
- Environment separation

Important:

> A Namespace is a logical Kubernetes boundary, not automatically a complete security boundary for every kind of resource or traffic.

---

# 33. Namespace Example

Imagine a company has:

```text
team-a
team-b
```

Both can use the same cluster:

```text
Kubernetes Cluster

team-a namespace
    |
    +-- frontend
    +-- backend
    +-- service

team-b namespace
    |
    +-- frontend
    +-- backend
    +-- service
```

They can use the same names inside different namespaces because names are typically scoped by namespace.

---

# 34. The Most Important Object Relationships

Memorize these relationships.

## Application

```text
Deployment
     |
     v
ReplicaSet
     |
     v
Pods
```

## Networking

```text
Service
     |
     v
Pods
```

## Node-level software

```text
DaemonSet
     |
     +---- Node 1 Pod
     +---- Node 2 Pod
     +---- Node 3 Pod
```

## Batch workloads

```text
Job
 |
 +---- Pod
```

## Organization

```text
Namespace
 |
 +---- Deployment
 +---- Service
 +---- Pod
 +---- Job
```

---

# 35. Declarative Kubernetes

One of Kubernetes' biggest ideas is **declarative configuration**.

Instead of saying:

```text
Start container A.
Start container B.
Start container C.
```

You say:

```yaml
replicas: 3
```

You declare:

> "I want three replicas."

Kubernetes continuously works toward that state.

This is why Kubernetes is often described as a:

> **Declarative system with reconciliation loops.**

---

# 36. Imperative vs Declarative

## Imperative

You tell the system what commands to execute.

Example:

```bash
kubectl create deployment my-app --image=my-app:v1
```

## Declarative

You define the desired state:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  replicas: 3
```

Then:

```bash
kubectl apply -f deployment.yaml
```

For real projects, declarative YAML is extremely common.

---

# 37. Labels

Labels are key-value metadata attached to objects.

Example:

```yaml
metadata:
  labels:
    app: backend
    environment: production
```

Labels are extremely important because Kubernetes uses them for selection.

For example:

```yaml
selector:
  matchLabels:
    app: backend
```

A Service can select Pods using labels.

```text
Service selector:
app=backend

        ↓

Pod 1: app=backend   ✓
Pod 2: app=backend   ✓
Pod 3: app=frontend  ✗
```

---

# 38. Selectors

Selectors allow Kubernetes resources to identify other resources.

Example:

```yaml
selector:
  matchLabels:
    app: backend
```

This means:

> Select objects having the label `app=backend`.

Selectors are fundamental to:

- Deployments
- ReplicaSets
- Services
- DaemonSets

---

# 39. Self-Healing Example

Suppose a Deployment has:

```yaml
replicas: 3
```

Current state:

```text
Pod A
Pod B
Pod C
```

Now Pod B crashes.

Actual state becomes:

```text
Pod A
Pod C
```

Desired state remains:

```text
3 Pods
```

The controller notices:

```text
Desired = 3
Actual  = 2
```

Then Kubernetes creates another Pod.

Eventually:

```text
Pod A
Pod C
Pod D
```

This is self-healing through reconciliation.

---

# 40. Scaling Example

Start with:

```yaml
replicas: 2
```

Then scale:

```bash
kubectl scale deployment backend --replicas=5
```

Kubernetes works toward:

```text
2 Pods
   ↓
5 Pods
```

You can also use autoscaling mechanisms such as the Horizontal Pod Autoscaler.

A typical idea is:

```text
CPU usage increases
       ↓
HPA decides more replicas are needed
       ↓
Deployment replica count increases
       ↓
ReplicaSet creates Pods
```

---

# 41. Rolling Update

Suppose:

```text
Current:
my-app:v1
```

You update to:

```text
my-app:v2
```

Deployment can gradually replace old Pods with new Pods.

Conceptually:

```text
v1 v1 v1
   ↓
v1 v1 v2
   ↓
v1 v2 v2
   ↓
v2 v2 v2
```

This reduces downtime and supports controlled application releases.

---

# 42. Rollback

If version 2 is broken, the Deployment can roll back to a previous revision.

Example:

```bash
kubectl rollout undo deployment my-app
```

Useful commands:

```bash
kubectl rollout status deployment my-app
kubectl rollout history deployment my-app
kubectl rollout undo deployment my-app
```

---

# 43. Health Checks

Kubernetes can use probes to determine application health.

Important probe types:

### Liveness Probe

Asks:

> "Is this container still alive?"

If it repeatedly fails, Kubernetes can restart the container.

### Readiness Probe

Asks:

> "Is this application ready to receive traffic?"

If not ready, the Pod can be removed from Service endpoints.

### Startup Probe

Useful for slow-starting applications.

It gives the application extra time to start before liveness checking becomes important.

---

# 44. Requests and Limits

Containers can specify resource requests and limits.

Example:

```yaml
resources:
  requests:
    cpu: "250m"
    memory: "256Mi"
  limits:
    cpu: "500m"
    memory: "512Mi"
```

### Request

The amount of resource Kubernetes uses when making scheduling decisions.

### Limit

The maximum resource boundary imposed by the runtime/kernel mechanisms.

This is important because the scheduler needs to know whether a node has enough capacity.

---

# 45. ConfigMap

A ConfigMap stores non-sensitive configuration.

Example:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  APP_MODE: "production"
```

Application configuration can then be injected into Pods.

---

# 46. Secret

A Secret is designed for sensitive configuration such as:

- Passwords
- Tokens
- API credentials
- Certificates

Important:

> Kubernetes Secret data is not automatically "encrypted everywhere" just because the object is called a Secret.

Security depends on cluster configuration, storage encryption, access control, and secret-management practices.

For production systems, external secret-management solutions may also be used.

---

# 47. Volumes and Persistent Storage

Containers are ephemeral.

If a container is replaced, its writable container filesystem should not be treated as permanent storage.

Kubernetes provides storage concepts such as:

- Volume
- PersistentVolume (PV)
- PersistentVolumeClaim (PVC)
- StorageClass

A simplified relationship:

```text
Pod
 |
 +--- PVC
       |
       v
      PV
       |
       v
Underlying Storage
```

---

# 48. Service Discovery

Kubernetes Services commonly work together with cluster DNS.

For example:

```text
backend Service
```

may be reachable internally using a DNS name such as:

```text
backend
```

or a fully qualified form based on the namespace and cluster domain.

This means applications can communicate using stable names instead of tracking changing Pod IPs.

---

# 49. A Real Application Example

Imagine a Spring Boot backend with three replicas.

Desired architecture:

```text
                    Internet
                       |
                       v
              LoadBalancer Service
                       |
                backend Service
                       |
          +------------+------------+
          |            |            |
       Pod 1          Pod 2        Pod 3
          ^            ^            ^
          |            |            |
          +-------- Deployment -----+
                       |
                  ReplicaSet
```

The Pods run on worker nodes.

```text
Worker Node 1
    └── Pod 1

Worker Node 2
    └── Pod 2

Worker Node 3
    └── Pod 3
```

Node-level components support their execution and networking:

```text
Worker Node
├── kubelet
├── container runtime
├── kube-proxy / networking implementation
└── Pod
```

---

# 50. Full Kubernetes Mental Model

Keep this picture in your head:

```text
                           USER
                            |
                            v
                         kubectl
                            |
                            v
                    +----------------+
                    | kube-apiserver |
                    +----------------+
                      /      |      \
                     /       |       \
                    v        v        v
                  etcd   scheduler  controllers
                                      |
                                      v
                             Desired state changes
                                      |
                     +----------------+----------------+
                     |                                 |
                     v                                 v
                 Worker 1                           Worker 2
                     |                                 |
                  kubelet                           kubelet
                     |                                 |
              Container Runtime                Container Runtime
                     |                                 |
                  Pods                              Pods
                     \                                 /
                      \---------- CNI ----------------/
                               |
                         Pod networking

Services provide stable
network access to selected Pods.
```

---

# 51. Quick Comparison Table

| Object / Component | Main Purpose |
|---|---|
| Pod | Smallest deployable workload unit |
| ReplicaSet | Maintains a desired number of matching Pods |
| Deployment | Manages stateless application rollouts and ReplicaSets |
| Service | Stable network endpoint for Pods |
| DaemonSet | Runs a Pod on each eligible node |
| Job | Runs a finite task to completion |
| Namespace | Logical grouping/isolation boundary for resources |
| StatefulSet | Manages stateful workloads with stable identity/storage |
| ConfigMap | Stores non-sensitive configuration |
| Secret | Stores sensitive configuration data |
| kube-apiserver | Kubernetes API entry point |
| etcd | Cluster state database/source of truth |
| kube-scheduler | Selects nodes for unscheduled Pods |
| kube-controller-manager | Runs reconciliation controllers |
| cloud-controller-manager | Integrates Kubernetes with cloud-provider APIs |
| kubelet | Node agent that ensures Pods run |
| Container Runtime | Runs/manages containers |
| kube-proxy | Implements Service networking rules in common setups |
| CNI | Provides Pod networking |
| kubectl | CLI client for interacting with Kubernetes |

---

# 52. What You Should Memorize First

Do not try to memorize every Kubernetes feature at once.

Start with these relationships:

```text
Kubernetes
├── Control Plane
│   ├── kube-apiserver
│   ├── etcd
│   ├── kube-scheduler
│   └── kube-controller-manager
│
└── Worker Node
    ├── kubelet
    ├── Container Runtime
    ├── kube-proxy / networking implementation
    └── Pods
```

Then:

```text
Deployment
    ↓
ReplicaSet
    ↓
Pods
```

```text
Service
    ↓
selects Pods
```

```text
DaemonSet
    ↓
one Pod on each eligible node
```

```text
Job
    ↓
run task → complete
```

```text
Namespace
    ↓
logical grouping of resources
```

And networking:

```text
CNI
    ↓
Pod networking
```

---

# 53. Final Mental Model

When you look at Kubernetes, ask five questions:

### 1. Who receives my request?

```text
kube-apiserver
```

### 2. Where is cluster state stored?

```text
etcd
```

### 3. Who decides which node gets a Pod?

```text
kube-scheduler
```

### 4. Who keeps desired state and actual state aligned?

```text
Controllers
```

### 5. Who actually runs the application?

```text
Worker Node
  |
  +-- kubelet
  +-- container runtime
  +-- Pod
```

Then add networking:

```text
CNI → Pod networking
Service → stable access to Pods
```

And workload management:

```text
Deployment → stateless applications
DaemonSet → one-per-node workloads
Job → finite tasks
Namespace → logical grouping
```

---

# 54. One Sentence for Every Important Concept

If you can explain these sentences without looking at notes, your Kubernetes foundation is strong:

- **Kubernetes** manages containerized workloads declaratively.
- **Control Plane** manages cluster state and makes scheduling/control decisions.
- **kube-apiserver** exposes the Kubernetes API.
- **etcd** stores cluster state.
- **kube-scheduler** selects nodes for unscheduled Pods.
- **kube-controller-manager** runs controllers that reconcile desired and actual state.
- **cloud-controller-manager** connects Kubernetes with cloud-provider APIs.
- **kubelet** is the node agent that ensures Pods run.
- **Container Runtime** actually runs containers.
- **kube-proxy** traditionally implements Service networking rules on nodes.
- **CNI** provides Pod networking.
- **kubectl** is the CLI used to talk to the API server.
- **Pod** is the smallest deployable unit.
- **ReplicaSet** maintains a desired number of Pods.
- **Deployment** manages stateless application releases and ReplicaSets.
- **Service** provides stable network access to Pods.
- **DaemonSet** runs a Pod on every eligible node.
- **Job** runs a finite task until completion.
- **Namespace** logically organizes and isolates resources.
- **Labels and selectors** connect Kubernetes resources to one another.

---

# 55. Recommended Learning Order After This

After understanding this architecture and the core objects, a strong hands-on sequence is:

```text
1. Pods
2. Labels + Selectors
3. Deployments
4. ReplicaSets
5. Services
6. Namespaces
7. ConfigMaps + Secrets
8. Probes
9. Resource Requests + Limits
10. Volumes + PVC
11. DaemonSets
12. Jobs + CronJobs
13. StatefulSets
14. Ingress
15. HPA
16. RBAC
17. NetworkPolicy
18. Scheduling
19. Helm
20. Kubernetes troubleshooting
```

The important progression is:

```text
Architecture
    ↓
Objects
    ↓
Networking
    ↓
Storage
    ↓
Security
    ↓
Scaling
    ↓
Scheduling
    ↓
Operations / Troubleshooting
```

That progression will give you a much stronger Kubernetes foundation than memorizing commands alone.
