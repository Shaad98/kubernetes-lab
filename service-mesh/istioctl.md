# istioctl

## 1. What is istioctl?

`istioctl` is the **command-line interface (CLI) for Istio**.

A simple comparison:

```text
kubectl
   ↓
Kubernetes CLI

istioctl
   ↓
Istio CLI
```

`istioctl` is primarily used to:

```text
Install Istio
Check Istio
Validate Istio configuration
Inspect Istio proxies
Debug Istio
Manage Istio configuration
```

---

# 2. istioctl vs Istio

These are NOT the same thing.

```text
istioctl
    ↓
CLI tool
    ↓
Runs on your machine
```

while:

```text
Istio
    ↓
Service mesh
    ↓
Runs inside/around your Kubernetes workloads
```

Think:

```text
YOUR MACHINE
────────────────────

istioctl
    │
    │ communicates with
    ▼
KUBERNETES CLUSTER
────────────────────

Istio
├── istiod
├── Envoy proxies
└── Istio configuration
```

---

# 3. Where is istioctl installed?

`istioctl` is installed on your workstation.

For your setup:

```bash
which istioctl
```

Expected:

```text
/home/shaad/istio-1.31.1/bin/istioctl
```

Check:

```bash
istioctl version
```

The important point:

**You do not install `istioctl` inside every Kubernetes node.**

It is a client-side CLI.

---

# 4. Installing Istio and getting istioctl

Download Istio:

```bash
curl -L https://istio.io/downloadIstio | sh -
```

This creates a directory similar to:

```text
istio-1.31.1/
```

Inside:

```text
istio-1.31.1/
├── bin/
│   └── istioctl
├── manifests/
├── samples/
└── ...
```

The `istioctl` executable is:

```text
bin/istioctl
```

---

# 5. Add istioctl to PATH

Temporary:

```bash
export PATH="$PATH:$HOME/istio-1.31.1/bin"
```

Permanent:

```bash
echo 'export PATH="$HOME/istio-1.31.1/bin:$PATH"' >> ~/.bashrc
source ~/.bashrc
```

Check:

```bash
which istioctl
```

and:

```bash
istioctl version
```

---

# 6. What happens when we run istioctl?

Suppose we run:

```bash
istioctl install
```

The command runs on our machine.

But the Istio installation happens in the Kubernetes cluster.

```text
Ubuntu Machine
│
└── istioctl
      │
      │ Kubernetes API
      ▼
Kubernetes Cluster
│
└── Istio
    └── istiod
```

So:

```text
istioctl = local tool

Istio = cluster/service-mesh components
```

---

# 7. Check Kubernetes context first

Before using `istioctl` to interact with Kubernetes, check the current context:

```bash
kubectl config current-context
```

For a kind cluster, it may be:

```text
kind-cws-cluster
```

This is important because `istioctl` works with the Kubernetes cluster configured through your Kubernetes configuration.

---

# 8. Pre-installation check

Before installing Istio:

```bash
istioctl x precheck
```

This performs checks to identify potential problems before installation.

Think:

```text
istioctl x precheck
        ↓
"Is this cluster ready for Istio?"
```

If problems are found, fix them before installing.

---

# 9. Installing Istio

One installation method is:

```bash
istioctl install
```

You can also provide an installation configuration.

For example:

```bash
istioctl install -f samples/bookinfo/demo-profile-no-gateways.yaml -y
```

Here:

```text
istioctl
    ↓
install Istio
    ↓
using specified configuration
    ↓
into Kubernetes cluster
```

---

# 10. What does `-f` mean?

```bash
-f
```

specifies a configuration file.

Example:

```bash
istioctl install \
  -f samples/bookinfo/demo-profile-no-gateways.yaml
```

The configuration file controls how Istio is installed.

---

# 11. What does `-y` mean?

```bash
-y
```

automatically confirms the installation.

Without `-y`, `istioctl` can ask for confirmation.

---

# 12. Istio installation profile

Istio provides different installation configurations/profiles.

Examples include:

```text
default
demo
minimal
```

A profile determines which components and settings are installed.

The `demo` profile is commonly used for learning and demonstrations because it provides a broader set of features.

Production installations normally require deliberate configuration rather than blindly using a demo setup.

---

# 13. Verify Istio installation

After installation:

```bash
kubectl get namespace
```

Look for:

```text
istio-system
```

Then:

```bash
kubectl get pods -n istio-system
```

You should see Istio components such as:

```text
istiod-xxxxx
```

You can also use:

```bash
istioctl verify-install
```

The exact resources depend on the Istio installation mode and profile.

---

# 14. What is istiod?

`istiod` is an important Istio control-plane component.

Simple mental model:

```text
                 istiod
              Control Plane
                   │
        ┌──────────┼──────────┐
        ▼          ▼          ▼
      Envoy      Envoy      Envoy
       Proxy      Proxy      Proxy
```

`istiod` provides configuration and control information to Istio's data-plane proxies.

---

# 15. What is Envoy?

Envoy is the proxy used in Istio's data plane.

An application Pod can have:

```text
Pod
├── Application
└── Envoy
```

The application handles business logic.

Envoy handles network-related service-mesh functionality.

Conceptually:

```text
Application
    │
    ▼
 Envoy
    │
    │ network traffic
    ▼
 Envoy
    │
    ▼
Application
```

---

# 16. Control plane vs data plane

This is one of the most important Istio concepts.

## Control plane

```text
istiod
```

It provides configuration/control functionality for the proxies.

## Data plane

```text
Envoy proxies
```

They handle actual service traffic.

Think:

```text
              CONTROL PLANE
                   │
                istiod
                   │
            configuration
                   │
       ┌───────────┼───────────┐
       ▼           ▼           ▼
     Envoy       Envoy       Envoy
       │           │           │
       └───────────┼───────────┘
                   │
              DATA PLANE
```

---

# 17. Sidecar injection

Istio can automatically inject an Envoy sidecar into Pods.

Label a namespace:

```bash
kubectl label namespace default istio-injection=enabled
```

Now new Pods in that namespace can receive an Envoy sidecar.

Without injection:

```text
Pod
└── Application
```

With injection:

```text
Pod
├── Application
└── Envoy
```

---

# 18. Why is the namespace label needed?

The label:

```text
istio-injection=enabled
```

tells Istio's injection mechanism:

> Automatically add the Istio sidecar to new Pods created in this namespace.

Important:

```text
Namespace label
    ≠
Install Istio
```

Istio must already be installed.

The label controls sidecar injection.

---

# 19. What happens when a Pod is created?

Suppose you deploy:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app
spec:
  replicas: 1
  template:
    spec:
      containers:
        - name: app
          image: my-app:v1
```

If sidecar injection is enabled, Istio's injection mechanism modifies the Pod specification so that an Envoy container is included.

Conceptually:

```text
Deployment
    │
    ▼
Pod creation request
    │
    ▼
Istio injection
    │
    ▼
Pod
├── Application
└── Envoy
```

---

# 20. Service-to-service communication

Suppose we have:

```text
document-service
       │
       ▼
chat-service
```

With Istio sidecars:

```text
┌─────────────────────┐
│ document Pod        │
│                     │
│ Document Service    │
│        │            │
│        ▼            │
│      Envoy          │
└────────┬────────────┘
         │
         │ network traffic
         ▼
┌─────────────────────┐
│ chat Pod            │
│                     │
│      Envoy          │
│        │            │
│        ▼            │
│ Chat Service        │
└─────────────────────┘
```

The Envoy proxies form the Istio data plane.

---

# 21. What can Istio provide?

Depending on configuration and mode, Istio can provide functionality such as:

```text
Traffic management
Retries
Timeouts
Load balancing
mTLS
Metrics
Tracing
Traffic policies
Service identity
```

This means applications don't necessarily need to implement every networking feature themselves.

---

# 22. mTLS

Istio can provide mutual TLS for service-to-service communication.

Normal TLS:

```text
Client ───── encrypted connection ─────> Server
```

mTLS provides authentication in both directions:

```text
Client <──── authenticated/encrypted ────> Server
```

With Istio proxies:

```text
Service A
    │
  Envoy
    │
    │ mTLS
    ▼
  Envoy
    │
Service B
```

The application itself can remain unaware of many of the transport-level security details.

---

# 23. Traffic management

Istio can control how traffic reaches service versions.

For example:

```text
chat-service
     │
     ├── v1
     └── v2
```

Traffic can be controlled using Istio configuration resources.

Important resources include:

```text
VirtualService
DestinationRule
Gateway
ServiceEntry
```

---

# 24. VirtualService

A `VirtualService` defines traffic-routing rules.

Conceptually:

```text
Client
  │
  ▼
VirtualService rules
  │
  ├── v1
  └── v2
```

For example, traffic could be configured to go to a particular service version.

---

# 25. DestinationRule

A `DestinationRule` defines policies for traffic to a destination.

It can define concepts such as:

```text
Subsets
Load balancing
Connection policies
Traffic policies
```

For example:

```text
chat-service
   │
   ├── v1 subset
   └── v2 subset
```

A `VirtualService` can then route traffic toward these subsets.

---

# 26. Istio Gateway

An Istio Gateway can handle traffic entering or leaving the mesh.

Conceptually:

```text
Internet
   │
   ▼
Istio Gateway
   │
   ▼
Service
```

This is different from the Kubernetes `Service` resource.

Istio also supports the Kubernetes Gateway API, which is a separate Kubernetes-native API for configuring gateways.

---

# 27. Istio vs Kubernetes Service

A Kubernetes Service provides stable network access to Pods.

```text
Client
   │
   ▼
Kubernetes Service
   │
   ├── Pod
   ├── Pod
   └── Pod
```

Istio adds service-mesh functionality around communication.

```text
Service A
   │
 Envoy
   │
   ▼
 Envoy
   │
Service B
```

Remember:

```text
Kubernetes Service
    ↓
Stable networking/load distribution to Pods

Istio
    ↓
Service communication management
```

They are complementary.

---

# 28. Istio vs Ingress

Ingress commonly handles traffic entering the cluster:

```text
Internet
   │
   ▼
Ingress
   │
   ▼
Service
   │
   ▼
Pod
```

Service mesh primarily focuses on service-to-service communication:

```text
Service A
   │
   ▼
Service B
   │
   ▼
Service C
```

Simple memory:

```text
Ingress
    ↓
Outside → Cluster

Service Mesh
    ↓
Service → Service
```

Istio can also provide gateway functionality, so the concepts can overlap.

---

# 29. Important istioctl commands

## Check version

```bash
istioctl version
```

## Find istioctl

```bash
which istioctl
```

## Pre-installation check

```bash
istioctl x precheck
```

## Install

```bash
istioctl install
```

## Install with configuration

```bash
istioctl install -f <file>
```

## Verify installation

```bash
istioctl verify-install
```

## Analyze configuration

```bash
istioctl analyze
```

`analyze` can help identify configuration problems in the mesh.

---

# 30. Proxy status

One useful diagnostic command is:

```bash
istioctl proxy-status
```

It shows information about Envoy proxies known to Istio.

Conceptually:

```text
Application Pods
      │
      ▼
Envoy proxies
      │
      ▼
istioctl proxy-status
```

This is useful when debugging whether proxies are connected and receiving configuration.

---

# 31. Proxy configuration

You can inspect a proxy's configuration using:

```bash
istioctl proxy-config
```

There are different subcommands for different configuration areas.

For example:

```bash
istioctl proxy-config cluster <pod-name>
istioctl proxy-config listener <pod-name>
istioctl proxy-config route <pod-name>
istioctl proxy-config endpoint <pod-name>
```

These commands help inspect what Envoy actually knows.

---

# 32. Why `istioctl` is useful for debugging

Suppose:

```text
Service A
   │
   ▼
Service B
```

but communication isn't working.

Kubernetes commands might tell you:

```bash
kubectl get pods
kubectl get svc
kubectl get endpoints
```

Istio adds additional questions:

```text
Is the Envoy proxy running?
Did Envoy receive configuration?
Is the route correct?
Are endpoints known?
Is mTLS causing the problem?
```

Useful commands include:

```bash
istioctl analyze
istioctl proxy-status
istioctl proxy-config
```

So:

```text
kubectl
   ↓
Debug Kubernetes

istioctl
   ↓
Debug Istio/service-mesh behavior
```

---

# 33. Istio installation flow

The overall flow is:

```text
1. Download Istio
       │
       ▼
2. Get istioctl
       │
       ▼
3. Add istioctl to PATH
       │
       ▼
4. Check Kubernetes context
       │
       ▼
5. istioctl x precheck
       │
       ▼
6. Install Istio
       │
       ▼
7. Verify istio-system
       │
       ▼
8. Enable sidecar injection
       │
       ▼
9. Deploy application
       │
       ▼
10. Application Pod receives Envoy
```

---

# 34. Local machine vs cluster

This distinction is extremely important.

## Local machine

```text
kubectl
helm
istioctl
```

These are CLI tools.

## Kubernetes cluster

```text
Istio
├── istiod
├── Envoy proxies
└── other required components
```

Therefore:

```text
istioctl install
```

does NOT mean:

> Install istioctl into Kubernetes.

It means:

> Use the local istioctl CLI to install Istio into the Kubernetes cluster.

---

# 35. istioctl vs Helm

They are separate tools.

```text
Helm
    ↓
General Kubernetes package manager

istioctl
    ↓
Istio-specific CLI
```

Istio can be installed using `istioctl`.

Istio can also be installed using Helm.

For learning Istio, `istioctl` provides a straightforward Istio-specific workflow.

---

# 36. Sidecar architecture

The traditional Istio sidecar model looks like:

```text
                ISTIO CONTROL PLANE
                       │
                    istiod
                       │
              configuration
                       │
       ┌───────────────┼───────────────┐
       ▼               ▼               ▼
┌─────────────┐  ┌─────────────┐  ┌─────────────┐
│ Pod         │  │ Pod         │  │ Pod         │
│             │  │             │  │             │
│ App         │  │ App         │  │ App         │
│             │  │             │  │             │
│ Envoy       │  │ Envoy       │  │ Envoy       │
└──────┬──────┘  └──────┬──────┘  └──────┬──────┘
       │                │                │
       └────────────────┼────────────────┘
                        │
                 Service traffic
```

---

# 37. Sidecar vs Ambient mode

Istio also supports **ambient mode**.

Traditional sidecar:

```text
Pod
├── Application
└── Envoy
```

Ambient mode changes the data-plane architecture so that a proxy does not necessarily have to be injected into every application Pod.

For learning the fundamentals, understand the sidecar model first:

```text
Application
    +
Envoy sidecar
```

Then learn ambient mode separately.

---

# 38. Most important mental model

Remember:

```text
istioctl
    ↓
Istio CLI

istiod
    ↓
Istio control plane

Envoy
    ↓
Istio data-plane proxy

Istio
    ↓
Service Mesh
```

Complete picture:

```text
YOUR MACHINE
────────────────────────

kubectl
helm
istioctl
   │
   │
   ▼
KUBERNETES CLUSTER
────────────────────────

              istiod
           Control Plane
                │
       ┌────────┼────────┐
       ▼        ▼        ▼
    Envoy     Envoy     Envoy
       │        │        │
       ▼        ▼        ▼
     App A    App B    App C
       │        │        │
       └────────┼────────┘
                │
        Service-to-service
           communication
```

---

# 39. Quick revision

```text
istioctl
    = Istio CLI

istioctl version
    = Check Istio CLI/version information

istioctl x precheck
    = Check cluster before installation

istioctl install
    = Install Istio into Kubernetes

istioctl verify-install
    = Verify Istio installation

istioctl analyze
    = Analyze Istio/Kubernetes configuration

istioctl proxy-status
    = Inspect Envoy proxy status

istioctl proxy-config
    = Inspect Envoy configuration

istiod
    = Istio control plane

Envoy
    = Istio data-plane proxy

Sidecar injection
    = Automatically add Envoy to application Pods

Istio
    = Service mesh
```
