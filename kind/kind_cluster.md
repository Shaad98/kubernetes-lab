# KIND Cluster Creation & Command Reference

This guide covers creating, inspecting, using, and deleting a multi-node Kubernetes cluster with [KIND (Kubernetes IN Docker)](https://kind.sigs.k8s.io/).

The example creates:

- 1 control-plane node
- 3 worker nodes
- Kubernetes `v1.31.2`
- Host port `80` mapped to a KIND node's port `80`
- Host port `443` mapped to a KIND node's port `443`

> KIND nodes are Docker/Podman/nerdctl containers. The worker nodes are useful for practicing scheduling, Deployments, Services, rolling updates, node labels, taints, and similar Kubernetes behavior, but they do not provide the same isolation or compute capacity as separate physical/VM nodes.

---

## 1. Prerequisites

Check that KIND, kubectl, and your container runtime are installed:

```bash
kind version
kubectl version --client

docker version
```

Check Docker is running:

```bash
docker ps
```

KIND can detect Docker, Podman, or nerdctl automatically. You can explicitly select a provider with `KIND_EXPERIMENTAL_PROVIDER` when needed.

---

## 2. Project Structure

A simple practice directory can look like:

```text
kind-cluster/
├── README.md
└── cluster.yaml
```

---

## 3. KIND Cluster Configuration

Create `cluster.yaml`:

```yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4

nodes:
  - role: control-plane
    image: kindest/node:v1.31.2

  - role: worker
    image: kindest/node:v1.31.2

  - role: worker
    image: kindest/node:v1.31.2

  - role: worker
    image: kindest/node:v1.31.2
    extraPortMappings:
      - containerPort: 80
        hostPort: 80
        protocol: TCP

      - containerPort: 443
        hostPort: 443
        protocol: TCP
```

### Important YAML syntax

Do not write the escaped characters from copied chat text such as `\-` or `\:`. The actual YAML should contain normal `-` and `:` characters.

---

## 4. Configuration Explained

### `kind`

```yaml
kind: Cluster
```

Tells KIND that this configuration describes a KIND cluster.

---

### `apiVersion`

```yaml
apiVersion: kind.x-k8s.io/v1alpha4
```

Specifies the KIND configuration API version.

This is the KIND configuration API version, not the Kubernetes API version used by your applications.

---

### `nodes`

```yaml
nodes:
```

Defines the containers that will become the Kubernetes nodes in the cluster.

Each `-` starts one node definition.

---

### `role`

```yaml
role: control-plane
```

Creates a Kubernetes control-plane node.

```yaml
role: worker
```

Creates a worker node.

A basic multi-node cluster therefore looks like:

```text
                 KIND Cluster
                      |
          +-----------+-----------+
          |           |           |
     Control Plane  Worker      Worker
                                  |
                                Worker
```

---

### `image`

```yaml
image: kindest/node:v1.31.2
```

Selects the node image, which determines the Kubernetes version bundled into the KIND node.

Here:

```text
kindest/node:v1.31.2
             ^^^^^^^^
             Kubernetes version
```

For reproducible environments, KIND also documents using an image digest (`@sha256:...`) instead of only a tag.

---

## 5. `extraPortMappings`

```yaml
extraPortMappings:
  - containerPort: 80
    hostPort: 80
    protocol: TCP
```

This forwards a port on the machine running KIND to a port on the KIND node container.

Think of it as:

```text
Your machine
localhost:80
      |
      | port mapping
      v
KIND node container :80
      |
      v
Kubernetes workload
```

### Attributes

| Attribute | Meaning | Example |
|---|---|---|
| `containerPort` | Port exposed on the KIND node container | `80` |
| `hostPort` | Port opened on the host machine | `80` |
| `listenAddress` | Host interface to bind to | `127.0.0.1` |
| `protocol` | Network protocol | `TCP`, `UDP`, `SCTP` |

Example with an explicit bind address:

```yaml
extraPortMappings:
  - containerPort: 80
    hostPort: 8080
    listenAddress: "127.0.0.1"
    protocol: TCP
```

Then:

```text
http://127.0.0.1:8080
        |
        v
KIND node :80
```

### Important

The host port must be available. If another process is already using port `80` or `443`, cluster creation can fail.

For a remote Linux server, remember that a port mapping only makes the port available on that host. External access also depends on the host firewall/security group.

---

## 6. Why Put the Port Mapping on a Specific Node?

In the example, mappings are attached to the last worker:

```yaml
- role: worker
  image: kindest/node:v1.31.2
  extraPortMappings:
    - containerPort: 80
      hostPort: 80
```

That means host traffic enters through that KIND node.

For an Ingress setup, you commonly map HTTP/HTTPS ports to a node and configure the Ingress controller accordingly.

---

# 7. Create the Cluster

From the directory containing `cluster.yaml`:

```bash
kind create cluster --name tws-cluster --config cluster.yaml
```

Recommended for scripts or CI-style practice:

```bash
kind create cluster \
  --name tws-cluster \
  --config cluster.yaml \
  --wait 5m
```

`--wait` makes KIND wait for the cluster to become ready, and it requires a duration such as `30s`, `2m`, or `5m`.

---

# 8. Verify the Cluster

### List KIND clusters

```bash
kind get clusters
```

Expected:

```text
tws-cluster
```

### Show cluster information

```bash
kubectl cluster-info --context kind-tws-cluster
```

> Important: the kubectl context is normally `kind-<cluster-name>`, so for `--name tws-cluster`, the context is `kind-tws-cluster`.

### Show current kubectl context

```bash
kubectl config current-context
```

### List all contexts

```bash
kubectl config get-contexts
```

### Switch context

```bash
kubectl config use-context kind-tws-cluster
```

### List nodes

```bash
kubectl get nodes
```

### More node information

```bash
kubectl get nodes -o wide
```

### Check system pods

```bash
kubectl get pods -A
```

### Check all namespaces

```bash
kubectl get namespaces
```

---

# 9. Useful `kind` Commands

## Create

```bash
kind create cluster
```

Create a default cluster named `kind`.

```bash
kind create cluster --name tws-cluster
```

Create a named cluster.

```bash
kind create cluster --name tws-cluster --config cluster.yaml
```

Create using a configuration file.

```bash
kind create cluster --name tws-cluster --image kindest/node:v1.31.2
```

Create with a specific node image when the same image is appropriate for the node setup.

```bash
kind create cluster --name tws-cluster --wait 60s
```

Wait until the cluster is ready or the timeout expires.

### Common create options

| Option | Purpose |
|---|---|
| `--name` | Cluster name |
| `--config` | KIND configuration YAML file |
| `--image` | Node image |
| `--wait` | Wait for readiness |
| `--kubeconfig` | Write/use a specific kubeconfig path |
| `--retain` | Retain nodes after creation failure for debugging |
| `--verbosity` | Increase command logging |
| `--dry-run` | Print generated configuration instead of creating the cluster |

See every option supported by your installed KIND version with:

```bash
kind create cluster --help
```

---

# 10. List Clusters

```bash
kind get clusters
```

Example:

```text
kind
tws-cluster
```

---

# 11. Get Cluster Nodes

KIND command:

```bash
kind get nodes --name tws-cluster
```

This shows the container/node names belonging to the cluster.

You can also use Docker:

```bash
docker ps --filter "label=io.x-k8s.kind.cluster=tws-cluster"
```

And Kubernetes:

```bash
kubectl get nodes
```

These answer slightly different questions:

```text
kind get nodes
    -> KIND node containers

kubectl get nodes
    -> Kubernetes nodes registered in the cluster
```

---

# 12. Delete the Cluster

```bash
kind delete cluster --name tws-cluster
```

Delete the default cluster:

```bash
kind delete cluster
```

### Confirm deletion

```bash
kind get clusters
```

You can also check Docker containers:

```bash
docker ps -a
```

---

# 13. Kubeconfig Commands

KIND automatically configures kubectl access after cluster creation.

View the current configuration:

```bash
kubectl config view
```

List contexts:

```bash
kubectl config get-contexts
```

Switch context:

```bash
kubectl config use-context kind-tws-cluster
```

Export KIND's kubeconfig to a file:

```bash
kind get kubeconfig --name tws-cluster > tws-cluster-kubeconfig.yaml
```

Use it temporarily:

```bash
KUBECONFIG=$PWD/tws-cluster-kubeconfig.yaml kubectl get nodes
```

> When creating a cluster, `kind create cluster --kubeconfig <path>` can be used when you want the cluster credentials written to a specific kubeconfig location.

---

# 14. Load a Local Docker Image into KIND

KIND uses container nodes, so an image available on your host is not automatically available inside the Kubernetes nodes.

Load an image:

```bash
kind load docker-image myapp:1.0 --name tws-cluster
```

List images inside a KIND node:

```bash
docker exec -it tws-cluster-worker crictl images
```

This is extremely useful when developing applications locally without pushing every image to Docker Hub.

Example workflow:

```bash
docker build -t myapp:1.0 .
kind load docker-image myapp:1.0 --name tws-cluster
kubectl create deployment myapp --image=myapp:1.0
```

For local images, make sure your Kubernetes workload does not unnecessarily try to pull the same image from a remote registry. For example, you can use:

```yaml
imagePullPolicy: IfNotPresent
```

---

# 15. Export Logs for Debugging

```bash
kind export logs --name tws-cluster
```

Specify a destination directory:

```bash
kind export logs ./kind-logs --name tws-cluster
```

This is useful when cluster creation fails or a node behaves unexpectedly.

---

# 16. Kubeconfig Details

KIND configures kubectl access automatically when the cluster is created. By default, the cluster credentials are stored in the kubeconfig used by kubectl (normally `${HOME}/.kube/config` when `KUBECONFIG` is not set).

Check the active kubeconfig/context:

```bash
kubectl config view
kubectl config current-context
kubectl config get-contexts
```

To create the cluster with a specific kubeconfig file, use the create-time flag:

```bash
kind create cluster \
  --name tws-cluster \
  --config cluster.yaml \
  --kubeconfig ./tws-cluster-kubeconfig.yaml
```

Then use that file explicitly:

```bash
KUBECONFIG=$PWD/tws-cluster-kubeconfig.yaml kubectl get nodes
```

---

# 17. Connect to a KIND Node

Find the node:

```bash
kind get nodes --name tws-cluster
```

Then inspect it using Docker:

```bash
docker exec -it tws-cluster-control-plane bash
```

For a worker:

```bash
docker exec -it tws-cluster-worker bash
```

The exact worker container names can be checked with:

```bash
kind get nodes --name tws-cluster
```

Inside a node you can inspect Kubernetes-related files, processes, networking, and container runtime state.

Exit with:

```bash
exit
```

---

# 18. Important `kind` Command Structure

Most KIND commands follow this structure:

```text
kind <COMMAND> <SUBCOMMAND> [FLAGS]
```

Examples:

```bash
kind create cluster
kind delete cluster
kind get clusters
kind get nodes
kind load docker-image
kind export logs
kind export kubeconfig
```

Display all top-level commands:

```bash
kind --help
```

Get help for a specific command:

```bash
kind create --help
kind create cluster --help
kind get --help
kind get clusters --help
kind load --help
kind load docker-image --help
kind export --help
```

This is the safest way to discover flags supported by the exact KIND version installed on your machine.

---

# 19. Useful `kubectl` Commands with KIND

Once the context is selected:

### Cluster

```bash
kubectl cluster-info
kubectl version
```

### Nodes

```bash
kubectl get nodes
kubectl get nodes -o wide
kubectl describe node <node-name>
```

### Pods

```bash
kubectl get pods
kubectl get pods -A
kubectl get pods -o wide
kubectl describe pod <pod-name>
```

### Deployments

```bash
kubectl get deployments
kubectl describe deployment <deployment-name>
kubectl rollout status deployment/<deployment-name>
kubectl rollout history deployment/<deployment-name>
```

### Services

```bash
kubectl get services
kubectl get svc
kubectl describe service <service-name>
```

### Namespaces

```bash
kubectl get namespaces
kubectl create namespace dev
kubectl config set-context --current --namespace=dev
```

### Logs

```bash
kubectl logs <pod-name>
kubectl logs -f <pod-name>
```

For a multi-container pod:

```bash
kubectl logs <pod-name> -c <container-name>
```

### Execute inside a pod

```bash
kubectl exec -it <pod-name> -- /bin/sh
```

If bash exists:

```bash
kubectl exec -it <pod-name> -- /bin/bash
```

### Events

```bash
kubectl get events --sort-by=.lastTimestamp
```

---

# 20. Test the HTTP Port Mapping

Suppose a workload or Ingress is listening on the KIND node's port `80`.

From the host:

```bash
curl http://localhost:80
```

For HTTPS:

```bash
curl -k https://localhost:443
```

The mapping itself does not automatically create an application or Ingress. Kubernetes still needs a workload and an appropriate Service/Ingress configuration behind the mapped port.

---

# 21. Node Labels

KIND supports extra labels on nodes through the configuration file.

Example:

```yaml
nodes:
  - role: worker
    labels:
      tier: frontend

  - role: worker
    labels:
      tier: backend
```

Then inspect labels:

```bash
kubectl get nodes --show-labels
```

Or:

```bash
kubectl get nodes -L tier
```

Use labels for scheduling:

```yaml
spec:
  nodeSelector:
    tier: backend
```

---

# 22. Extra Mounts

KIND can mount a host directory into a node container.

Example:

```yaml
nodes:
  - role: worker
    extraMounts:
      - hostPath: /tmp/kind-data
        containerPath: /data
```

Useful attributes include:

| Attribute | Meaning |
|---|---|
| `hostPath` | Directory/file on the host |
| `containerPath` | Path inside the KIND node |
| `readOnly` | Mount read-only when `true` |
| `selinuxRelabel` | Apply SELinux relabeling when needed |
| `propagation` | Mount propagation mode |

---

# 23. Networking Options

KIND also supports cluster-wide networking configuration.

Example:

```yaml
networking:
  ipFamily: ipv4
```

Supported IP family modes include:

```yaml
networking:
  ipFamily: ipv4
```

```yaml
networking:
  ipFamily: ipv6
```

```yaml
networking:
  ipFamily: dual
```

You can also customize the API server address/port, but exposing the KIND API server beyond loopback should be treated carefully.

Example:

```yaml
networking:
  apiServerAddress: "127.0.0.1"
  apiServerPort: 6443
```

---

# 24. Advanced Configuration Attributes

The KIND configuration API supports additional cluster/node settings.

## Cluster-wide

```yaml
name: my-cluster

featureGates:
  SomeFeatureGate: true

runtimeConfig:
  "api/alpha": "false"

networking:
  ipFamily: ipv4
  apiServerAddress: "127.0.0.1"
  apiServerPort: 6443
```

## Per-node

```yaml
nodes:
  - role: control-plane
    image: kindest/node:v1.31.2
    extraMounts: []
    extraPortMappings: []
    labels: {}
    kubeadmConfigPatches: []
```

Not every option is required. Start with `role`, then add only the configuration your experiment needs.

---

# 25. Full Example for Practice

```yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4

name: tws-cluster

networking:
  ipFamily: ipv4

nodes:
  - role: control-plane
    image: kindest/node:v1.31.2

  - role: worker
    image: kindest/node:v1.31.2
    labels:
      tier: frontend

  - role: worker
    image: kindest/node:v1.31.2
    labels:
      tier: backend

  - role: worker
    image: kindest/node:v1.31.2
    labels:
      tier: backend
    extraPortMappings:
      - containerPort: 80
        hostPort: 80
        listenAddress: "127.0.0.1"
        protocol: TCP
      - containerPort: 443
        hostPort: 443
        listenAddress: "127.0.0.1"
        protocol: TCP
```

Create it:

```bash
kind create cluster --config cluster.yaml --wait 5m
```

Verify it:

```bash
kind get clusters
kubectl get nodes -o wide
kubectl get pods -A
kubectl config current-context
```

Delete it:

```bash
kind delete cluster --name tws-cluster
```

---

# 26. Daily Practice Command Set

These are the commands worth memorizing first:

```bash
# Create
kind create cluster --name tws-cluster --config cluster.yaml

# List
kind get clusters

# Nodes managed by KIND
kind get nodes --name tws-cluster

# Cluster information
kubectl cluster-info --context kind-tws-cluster

# Kubernetes nodes
kubectl get nodes -o wide

# All pods
kubectl get pods -A

# Current context
kubectl config current-context

# Available contexts
kubectl config get-contexts

# Switch context
kubectl config use-context kind-tws-cluster

# Load a locally built image
kind load docker-image myapp:1.0 --name tws-cluster

# Export logs
kind export logs ./kind-logs --name tws-cluster

# Delete
kind delete cluster --name tws-cluster
```

---

# 27. Troubleshooting

## Port 80/443 already in use

Check the port:

```bash
sudo ss -ltnp | grep ':80 '
sudo ss -ltnp | grep ':443 '
```

Or choose different host ports:

```yaml
extraPortMappings:
  - containerPort: 80
    hostPort: 8080
    protocol: TCP
```

Then access:

```text
http://localhost:8080
```

## Wrong kubectl context

Check:

```bash
kubectl config get-contexts
```

Switch:

```bash
kubectl config use-context kind-tws-cluster
```

## Cluster creation failed

Create with retained nodes so you can investigate:

```bash
kind create cluster --name tws-cluster --config cluster.yaml --retain
```

Export logs:

```bash
kind export logs ./kind-logs --name tws-cluster
```

## Docker is not accessible

```bash
docker ps
```

If Docker is not running, start/fix the Docker service before creating the KIND cluster.

## Local image cannot be pulled

Load the image into KIND:

```bash
kind load docker-image myapp:1.0 --name tws-cluster
```

---

# 28. Quick Mental Model

Remember KIND as:

```text
                  Your Machine
                       |
                    Docker
                       |
       +---------------+----------------+
       |               |                |
       v               v                v
 control-plane     worker-1         worker-2 ...
       |
       +---- Kubernetes control-plane components
                       |
                       v
                  Kubernetes API
                       |
                    kubectl
```

KIND is therefore excellent for learning Kubernetes locally while keeping the setup lightweight compared with building a full multi-VM Kubernetes cluster.

---

## References

- KIND Quick Start: https://kind.sigs.k8s.io/docs/user/quick-start/
- KIND Configuration: https://kind.sigs.k8s.io/docs/user/configuration/
- KIND Documentation: https://kind.sigs.k8s.io/