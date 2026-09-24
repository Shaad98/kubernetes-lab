# Helm

## 1. What is Helm?

**Helm is a package manager for Kubernetes.**

A package manager helps us install, configure, upgrade, and remove applications.

For example, on Ubuntu we have:

```text
apt
```

For Python:

```text
pip
```

For Node.js:

```text
npm
```

For Kubernetes:

```text
Helm
```

The important difference is that Helm manages **Kubernetes applications**, which can consist of many Kubernetes resources.

---

# 2. Why do we need Helm?

Suppose we want to deploy an application manually.

We might have:

```text
deployment.yaml
service.yaml
configmap.yaml
secret.yaml
ingress.yaml
serviceaccount.yaml
role.yaml
rolebinding.yaml
```

We could run:

```bash
kubectl apply -f deployment.yaml
kubectl apply -f service.yaml
kubectl apply -f configmap.yaml
kubectl apply -f secret.yaml
kubectl apply -f ingress.yaml
...
```

This works.

But managing a large application with many YAML files can become difficult.

Helm allows these Kubernetes resources to be packaged together.

```text
                    Helm Chart
                        │
          ┌─────────────┼─────────────┐
          ▼             ▼             ▼
     Deployment      Service       ConfigMap
          │             │             │
          └─────────────┼─────────────┘
                        ▼
                 Kubernetes Cluster
```

---

# 3. Helm is NOT Kubernetes

This is important.

Helm is a **tool used to manage Kubernetes applications**.

```text
Kubernetes
    ↓
Runs containers and manages resources

Helm
    ↓
Packages and manages Kubernetes applications
```

Helm does not replace:

```bash
kubectl
```

They have different purposes.

---

# 4. Helm vs kubectl

## kubectl

`kubectl` works directly with Kubernetes resources.

Example:

```bash
kubectl apply -f deployment.yaml
```

You are saying:

> Create/update this Kubernetes resource.

---

## Helm

Helm works with a **Chart**.

Example:

```bash
helm install my-app ./my-chart
```

You are saying:

> Install this packaged Kubernetes application.

Conceptually:

```text
kubectl
   ↓
Kubernetes resources

Helm
   ↓
Chart
   ↓
Multiple Kubernetes resources
```

---

# 5. What is a Helm Chart?

A **Chart** is a package containing everything needed to define a Kubernetes application.

A typical chart looks like:

```text
my-app/
├── Chart.yaml
├── values.yaml
├── templates/
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── configmap.yaml
│   └── ingress.yaml
└── charts/
```

---

# 6. Chart.yaml

`Chart.yaml` contains metadata about the chart.

Example:

```yaml
apiVersion: v2
name: my-app
description: My Kubernetes application
type: application
version: 1.0.0
appVersion: "1.0"
```

Important fields:

```text
name
    Name of the chart

version
    Version of the chart

appVersion
    Version of the application
```

---

# 7. values.yaml

`values.yaml` contains configurable values.

Example:

```yaml
replicaCount: 2

image:
  repository: nginx
  tag: "1.27"

service:
  type: ClusterIP
  port: 80
```

Instead of hardcoding these values inside Kubernetes YAML, templates can use them.

For example:

```yaml
replicas: {{ .Values.replicaCount }}
```

and:

```yaml
image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
```

This is one of the biggest benefits of Helm.

---

# 8. Templates

Templates are Kubernetes YAML files containing variables.

Example:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Release.Name }}-app
spec:
  replicas: {{ .Values.replicaCount }}
  template:
    spec:
      containers:
        - name: app
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
```

Helm processes the template and generates normal Kubernetes YAML.

Conceptually:

```text
values.yaml
     +
templates/
     ↓
   Helm
     ↓
Rendered Kubernetes YAML
     ↓
Kubernetes API
```

---

# 9. Why values are useful

Suppose we want:

### Development

```yaml
replicaCount: 1
```

### Production

```yaml
replicaCount: 5
```

We don't need to rewrite the Deployment.

We can provide different values.

```text
                    Same Chart
                       │
              ┌────────┴────────┐
              ▼                 ▼
        dev values          prod values
        replicas: 1         replicas: 5
              │                 │
              ▼                 ▼
          Dev release       Prod release
```

---

# 10. What is a Helm Release?

A **release** is an installed instance of a Helm chart.

For example:

```bash
helm install my-app ./my-chart
```

Here:

```text
Chart:
./my-chart

Release:
my-app
```

Think:

```text
Chart = Package

Release = Installed instance of that package
```

The same chart can be installed multiple times.

```bash
helm install dev ./my-chart
helm install staging ./my-chart
helm install production ./my-chart
```

Now we have:

```text
Chart
 │
 ├── dev release
 ├── staging release
 └── production release
```

---

# 11. Helm Repository

A Helm repository stores Helm charts.

Similar idea:

```text
Docker Hub
    ↓
Container images

Helm Repository
    ↓
Helm charts
```

You can add a repository:

```bash
helm repo add <repo-name> <repository-url>
```

Example:

```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
```

Then:

```bash
helm repo update
```

Search:

```bash
helm search repo nginx
```

---

# 12. Installing Helm

Helm itself is installed on your local machine.

Check:

```bash
which helm
```

Example:

```text
/usr/local/bin/helm
```

Check version:

```bash
helm version
```

Your Helm installation is therefore:

```text
Ubuntu Machine
│
└── /usr/local/bin/helm
```

Helm does not need to be installed inside every Kubernetes node.

---

# 13. How Helm communicates with Kubernetes

Suppose you run:

```bash
helm install my-app ./my-chart
```

The general flow is:

```text
Your terminal
     │
     ▼
   Helm
     │
     │ Kubernetes configuration/context
     ▼
Kubernetes API Server
     │
     ▼
Kubernetes resources
```

Check your current Kubernetes context:

```bash
kubectl config current-context
```

For a kind cluster:

```text
kind-cws-cluster
```

Helm uses the Kubernetes configuration to determine which cluster it is working with.

---

# 14. Creating a Helm Chart

Create a new chart:

```bash
helm create my-app
```

Helm generates:

```text
my-app/
├── Chart.yaml
├── values.yaml
├── charts/
├── templates/
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── ingress.yaml
│   ├── serviceaccount.yaml
│   ├── _helpers.tpl
│   └── tests/
└── .helmignore
```

This gives us a starting point for a Helm application.

---

# 15. Inspect a Chart

Show chart information:

```bash
helm show chart ./my-app
```

Show default values:

```bash
helm show values ./my-app
```

Show all available information:

```bash
helm show all ./my-app
```

---

# 16. Render templates without installing

One extremely useful command:

```bash
helm template my-app ./my-app
```

This renders the Helm templates into Kubernetes YAML.

It does not install the release.

Flow:

```text
Chart
  +
values
  ↓
helm template
  ↓
Generated Kubernetes YAML
```

This is very useful for debugging.

---

# 17. Install a Chart

Example:

```bash
helm install my-app ./my-app
```

Here:

```text
my-app
    ↓
Release name

./my-app
    ↓
Chart
```

Check releases:

```bash
helm list
```

---

# 18. Install into a namespace

Example:

```bash
helm install my-app ./my-app \
  --namespace my-namespace \
  --create-namespace
```

Now the release is installed into:

```text
my-namespace
```

---

# 19. Override values

Suppose `values.yaml` contains:

```yaml
replicaCount: 1
```

We can override it:

```bash
helm install my-app ./my-app \
  --set replicaCount=3
```

Now:

```text
Default:
replicaCount = 1

Override:
replicaCount = 3
```

---

# 20. Using a custom values file

Create:

```text
values-dev.yaml
```

Example:

```yaml
replicaCount: 1

image:
  tag: "dev"
```

Then:

```bash
helm install my-app ./my-app \
  -f values-dev.yaml
```

For production:

```bash
helm install my-app ./my-app \
  -f values-prod.yaml
```

Same chart, different configuration.

---

# 21. Upgrade a Release

Suppose we already installed:

```bash
helm install my-app ./my-app
```

Then we change the chart or values.

Use:

```bash
helm upgrade my-app ./my-app
```

Flow:

```text
Existing Release
       │
       │ helm upgrade
       ▼
New Chart/Values
       │
       ▼
Updated Kubernetes resources
```

---

# 22. Rollback

Helm keeps release revisions.

Check history:

```bash
helm history my-app
```

Example:

```text
REVISION
1
2
3
```

Rollback:

```bash
helm rollback my-app 2
```

This returns the release to revision 2.

---

# 23. Uninstall

Remove a release:

```bash
helm uninstall my-app
```

The Helm release is removed and the Kubernetes resources managed by that release are generally removed as well.

---

# 24. Useful Helm commands

## Version

```bash
helm version
```

## List releases

```bash
helm list
```

## List releases in all namespaces

```bash
helm list -A
```

## Add repository

```bash
helm repo add <name> <url>
```

## Update repositories

```bash
helm repo update
```

## Search

```bash
helm search repo <keyword>
```

## Create chart

```bash
helm create <chart-name>
```

## Install

```bash
helm install <release> <chart>
```

## Upgrade

```bash
helm upgrade <release> <chart>
```

## History

```bash
helm history <release>
```

## Rollback

```bash
helm rollback <release> <revision>
```

## Uninstall

```bash
helm uninstall <release>
```

## Render templates

```bash
helm template <release> <chart>
```

## Show values

```bash
helm show values <chart>
```

---

# 25. Helm architecture — simple mental model

Think of Helm as:

```text
                 YOUR MACHINE
              ┌───────────────┐
              │     Helm      │
              └───────┬───────┘
                      │
                      │
                      ▼
              Kubernetes API
                      │
          ┌───────────┼───────────┐
          ▼           ▼           ▼
      Deployment    Service    ConfigMap
```

The important point:

**Helm is a tool for packaging and managing Kubernetes applications.**

---

# 26. Helm does not run your application

This is another important distinction.

Helm does not:

```text
❌ Run containers
❌ Schedule Pods
❌ Replace kubelet
❌ Replace Kubernetes
```

Kubernetes does those things.

Helm helps create/manage the Kubernetes resources that make up an application.

```text
Helm
  ↓
Creates/manages Kubernetes resources
  ↓
Kubernetes
  ↓
Runs the application
```

---

# 27. Helm and YAML

Without Helm:

```text
deployment.yaml
service.yaml
configmap.yaml
```

With Helm:

```text
Chart
├── templates/
│   ├── deployment.yaml
│   ├── service.yaml
│   └── configmap.yaml
│
└── values.yaml
```

Helm templates are still ultimately used to produce Kubernetes resources.

So learning Kubernetes YAML first is extremely useful before learning Helm.

---

# 28. Helm and Istio

Helm and Istio are separate concepts.

```text
Helm
  ↓
Kubernetes package manager

Istio
  ↓
Service mesh
```

Helm can be used to install Istio, but Helm is not part of Istio.

Similarly, Helm can install many other applications:

```text
Helm
├── Istio
├── Prometheus
├── Grafana
├── Redis
├── PostgreSQL
├── Kafka
└── many others
```

---

# 29. Final mental model

Remember these four words:

```text
Chart
  ↓
Package

Release
  ↓
Installed instance

values.yaml
  ↓
Configuration

Helm
  ↓
Manages Charts and Releases
```

The complete flow:

```text
Helm Chart
    │
    ├── Templates
    ├── values.yaml
    └── Chart.yaml
          │
          ▼
        Helm
          │
          ▼
   Kubernetes API Server
          │
          ▼
 Kubernetes Resources
          │
          ▼
        Pods
```

---

# 30. Quick revision

```text
Helm
    = Kubernetes package manager

Chart
    = Package containing Kubernetes application definitions

Release
    = Installed instance of a Chart

values.yaml
    = Default configurable values

templates/
    = Kubernetes YAML templates

helm install
    = Install a Chart

helm upgrade
    = Update a Release

helm rollback
    = Return to an earlier Release revision

helm uninstall
    = Remove a Release

helm template
    = Render templates without installing
```
