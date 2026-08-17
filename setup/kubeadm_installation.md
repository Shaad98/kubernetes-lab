# Kubeadm Kubernetes Cluster Setup

This guide explains how to create a Kubernetes cluster manually using **kubeadm** on Ubuntu EC2 instances.

The cluster consists of:

```text
                    AWS VPC
                       |
          +------------+------------+
          |                         |
   Control Plane                Worker Nodes
   -------------                ------------
   kube-apiserver               kubelet
   etcd                         containerd
   scheduler                    kube-proxy
   controller-manager
          |
          +-------------------------------+
                                          |
                              Kubernetes API :6443
```

## Cluster Architecture

For this lab, we use:

* **1 Control Plane node**
* **2 Worker nodes**
* Ubuntu Linux
* containerd as the container runtime
* kubeadm to bootstrap the cluster
* kubelet to run Kubernetes workloads on each node
* kubectl to communicate with the cluster
* Calico as the CNI (Container Network Interface)

---

# 1. Prerequisites

Each EC2 instance should have:

* Ubuntu Linux
* At least **2 GB RAM**
* At least **2 CPUs recommended for the control-plane**
* Internet connectivity
* `sudo` access
* Unique hostname for every node
* Network connectivity between all nodes

Kubernetes officially requires at least 2 GB RAM per machine and recommends 2 CPUs or more for control-plane machines.

For this learning lab, AWS `t2.medium` or a similar instance is sufficient.

> **Important:** `t2.medium` is not a Kubernetes requirement. It is simply an AWS instance size suitable for this lab.

---

# 2. AWS EC2 Setup

Create three EC2 instances:

```text
Control Plane
    |
    +-- Worker 1
    |
    +-- Worker 2
```

All instances should:

* Be in the **same VPC**
* Be able to communicate with each other
* Use the same Security Group

Using the same Security Group makes the lab easier to configure.

---

# 3. AWS Security Group

The Security Group controls network traffic reaching your EC2 instances.

## SSH

Allow:

```text
Protocol: TCP
Port: 22
Source: Your IP address
```

This allows you to SSH into the instances.

Avoid using:

```text
0.0.0.0/0
```

for SSH unless you specifically need it for a temporary lab.

---

## Kubernetes API Server

Allow:

```text
Protocol: TCP
Port: 6443
Source: Kubernetes nodes / trusted network
```

Port `6443` is the Kubernetes API Server port.

Worker nodes communicate with the control plane through the Kubernetes API Server.

The official Kubernetes port documentation lists `6443/TCP` for the API server.

---

## Other Kubernetes Ports

For a proper multi-node cluster, Kubernetes uses additional ports.

### Control Plane

| Port      | Protocol | Purpose               |
| --------- | -------- | --------------------- |
| 6443      | TCP      | Kubernetes API Server |
| 2379-2380 | TCP      | etcd                  |
| 10250     | TCP      | Kubelet API           |
| 10257     | TCP      | Controller Manager    |
| 10259     | TCP      | Scheduler             |

### Worker Nodes

| Port        | Protocol | Purpose           |
| ----------- | -------- | ----------------- |
| 10250       | TCP      | Kubelet API       |
| 10256       | TCP      | kube-proxy        |
| 30000-32767 | TCP/UDP  | NodePort Services |

These are the standard Kubernetes ports.

### For this AWS lab

You should **not expose all of these ports to the entire internet**.

A better Security Group design is:

```text
Internet
   |
   +---- TCP 22 ----> EC2 nodes
   |
   +---- TCP 6443 --> Control Plane
```

And for Kubernetes internal communication:

```text
Control Plane <------> Worker Nodes
       |
       +---- Kubernetes internal ports
```

Allow the internal Kubernetes ports using the **Security Group itself as the source**, or restrict them to the private CIDR of your VPC.

This is much safer than:

```text
0.0.0.0/0
```

for every Kubernetes port.

---

# 4. Check Hostnames

Run this on every node:

```bash
hostname
```

Each node should have a unique hostname.

For example:

```text
control-plane
worker-1
worker-2
```

You can change the hostname with:

```bash
sudo hostnamectl set-hostname control-plane
```

On worker nodes:

```bash
sudo hostnamectl set-hostname worker-1
```

and:

```bash
sudo hostnamectl set-hostname worker-2
```

### Why?

Kubernetes needs each node to be uniquely identifiable.

---

# 5. Run the Following Steps on ALL Nodes

Everything in this section must be performed on:

```text
Control Plane
Worker 1
Worker 2
```

---

## Step 5.1 — Disable Swap

Run:

```bash
sudo swapoff -a
```

### What does this do?

Swap allows Linux to move memory pages from RAM to disk.

Kubernetes expects predictable memory management and therefore traditionally requires swap to be disabled for kubeadm-based setups.

Check:

```bash
free -h
```

The `Swap` value should show `0B` or no active swap.

> `swapoff -a` disables swap immediately, but it does not necessarily prevent swap from being enabled again after reboot. For a persistent setup, remove or comment out the swap entry in `/etc/fstab`.

---

# 6. Load Required Kernel Modules

Run:

```bash
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF
```

Then:

```bash
sudo modprobe overlay
sudo modprobe br_netfilter
```

### What are these?

`overlay`:

Used by container runtimes for the OverlayFS filesystem.

`br_netfilter`:

Allows Linux bridge traffic to be processed by networking/firewall rules.

Kubernetes networking relies on these kernel capabilities.

Verify:

```bash
lsmod | grep overlay
lsmod | grep br_netfilter
```

You should see both modules.

---

# 7. Configure Linux Networking

Create the Kubernetes sysctl configuration:

```bash
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward = 1
EOF
```

Apply the configuration:

```bash
sudo sysctl --system
```

### What does this do?

The important setting is:

```text
net.ipv4.ip_forward = 1
```

It allows the Linux machine to forward packets between network interfaces.

This is important because a Kubernetes node may need to forward traffic between:

```text
Pod network
     |
     v
Node network
     |
     v
Other nodes
```

Check:

```bash
sysctl net.ipv4.ip_forward
```

Expected:

```text
net.ipv4.ip_forward = 1
```

---

# 8. Install containerd

Kubernetes needs a **container runtime** to actually run containers.

We will use:

```text
containerd
```

Kubernetes itself does not run containers directly.

The relationship is roughly:

```text
kubelet
   |
   | CRI
   v
containerd
   |
   v
containers
```

---

## Step 8.1 — Install prerequisites

```bash
sudo apt-get update
sudo apt-get install -y ca-certificates curl gpg
```

---

## Step 8.2 — Add Docker repository

containerd is provided through Docker's package repository.

Create the keyring directory:

```bash
sudo install -m 0755 -d /etc/apt/keyrings
```

Download the repository key:

```bash
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg \
  -o /etc/apt/keyrings/docker.asc
```

Set permissions:

```bash
sudo chmod a+r /etc/apt/keyrings/docker.asc
```

Add the repository:

```bash
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo "$VERSION_CODENAME") stable" \
  | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

Update package information:

```bash
sudo apt-get update
```

---

## Step 8.3 — Install containerd

```bash
sudo apt-get install -y containerd.io
```

Verify:

```bash
containerd --version
```

---

# 9. Configure containerd

Generate the default configuration:

```bash
containerd config default | sudo tee /etc/containerd/config.toml
```

Now configure containerd to use the `systemd` cgroup driver.

```bash
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
```

Restart containerd:

```bash
sudo systemctl restart containerd
```

Enable it at boot:

```bash
sudo systemctl enable containerd
```

Check its status:

```bash
sudo systemctl status containerd
```

You should see:

```text
Active: active (running)
```

### Why `SystemdCgroup = true`?

Linux systems using systemd should use the `systemd` cgroup driver for the container runtime.

The kubelet and container runtime need matching cgroup drivers. Kubernetes recommends `systemd` for kubeadm-based setups.

Also, modern kubeadm defaults the kubelet to `systemd`, so configuring containerd to use the same driver avoids mismatches.

---

# 10. Install Kubernetes Components

We need three Kubernetes tools:

```text
kubeadm
kubelet
kubectl
```

### kubeadm

Used to create and manage the Kubernetes cluster.

### kubelet

Runs on every node and communicates with the Kubernetes control plane.

### kubectl

Command-line tool used to interact with the Kubernetes cluster.

---

## Step 10.1 — Install repository prerequisites

```bash
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl gpg
```

Create the keyring directory:

```bash
sudo mkdir -p -m 755 /etc/apt/keyrings
```

---

## Step 10.2 — Add Kubernetes repository

Kubernetes now uses the `pkgs.k8s.io` repositories.

The old `apt.kubernetes.io` repository is deprecated and should not be used for new installations.

For example, for Kubernetes **v1.34**:

```bash
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.34/deb/Release.key \
  | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
```

Add the repository:

```bash
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.34/deb/ /' \
  | sudo tee /etc/apt/sources.list.d/kubernetes.list
```

Update:

```bash
sudo apt-get update
```

Install:

```bash
sudo apt-get install -y kubelet kubeadm kubectl
```

Prevent accidental automatic upgrades:

```bash
sudo apt-mark hold kubelet kubeadm kubectl
```

The Kubernetes repository is minor-version specific, so all nodes in the cluster should use compatible Kubernetes versions.

Check:

```bash
kubeadm version
kubectl version --client
kubelet --version
```

---

# 11. Initialize the Control Plane

Now switch to the **control-plane node only**.

Run:

```bash
sudo kubeadm init --pod-network-cidr=192.168.0.0/16
```

### What does this do?

`kubeadm init` creates the Kubernetes control plane.

It sets up components such as:

```text
kube-apiserver
etcd
kube-scheduler
kube-controller-manager
```

It also creates the certificates and configuration required by the cluster.

The `--pod-network-cidr` defines the IP range used for Kubernetes Pods.

Here we use:

```text
192.168.0.0/16
```

This matches the default Calico networking configuration used in the documented setup. Calico's current documentation notes that the Pod CIDR should be consistent with the chosen network configuration.

---

# 12. Configure kubectl

After `kubeadm init` completes, configure `kubectl` for your normal user.

Run:

```bash
mkdir -p "$HOME/.kube"
```

Copy the admin kubeconfig:

```bash
sudo cp -i /etc/kubernetes/admin.conf "$HOME/.kube/config"
```

Change ownership:

```bash
sudo chown "$(id -u)":"$(id -g)" "$HOME/.kube/config"
```

Now test:

```bash
kubectl get nodes
```

You will probably see:

```text
control-plane   NotReady
```

### Why NotReady?

Because we have not installed the Kubernetes network plugin yet.

This is expected.

---

# 13. Install Calico CNI

Kubernetes needs a **CNI plugin** to provide networking between Pods.

We will use:

```text
Calico
```

Current Calico documentation recommends the operator-based installation for new clusters.

Install the Calico CRDs:

```bash
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.32.1/manifests/v1_crd_projectcalico_org.yaml
```

Install the Tigera operator:

```bash
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.32.1/manifests/tigera-operator.yaml
```

Install Calico:

```bash
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.32.1/manifests/custom-resources.yaml
```

Check:

```bash
kubectl get pods -A
```

You can also monitor Calico:

```bash
watch kubectl get tigerastatus
```

Wait until the Calico components become available.

---

# 14. Generate Worker Join Command

On the **control-plane node**:

```bash
kubeadm token create --print-join-command
```

You will get something similar to:

```bash
kubeadm join 10.0.1.10:6443 \
  --token abcdef.0123456789abcdef \
  --discovery-token-ca-cert-hash sha256:xxxxxxxxxxxxxxxx
```

This command contains:

* Control-plane private IP
* API Server port
* Authentication token
* Cluster CA hash

The generated command is what a worker uses to securely discover and join the cluster.

---

# 15. Join Worker Nodes

Switch to **Worker 1**.

Before joining, if this is a previously initialized node, reset it:

```bash
sudo kubeadm reset -f
```

> The original command `sudo kubeadm reset pre-flight checks` was incorrect. `reset` is the kubeadm subcommand; `pre-flight checks` is not a valid argument sequence.

For a fresh worker node, you normally do not need `kubeadm reset`.

Now paste the join command generated on the control plane.

Example:

```bash
sudo kubeadm join 10.0.1.10:6443 \
  --token abcdef.0123456789abcdef \
  --discovery-token-ca-cert-hash sha256:xxxxxxxxxxxxxxxx
```

Do the same on Worker 2.

---

# 16. About `--cri-socket`

You may see commands such as:

```bash
--cri-socket unix:///run/containerd/containerd.sock
```

You generally **do not need to specify this** when containerd is the only installed CRI runtime and kubeadm can detect it.

If kubeadm cannot automatically detect the runtime, you can explicitly specify:

```bash
--cri-socket unix:///run/containerd/containerd.sock
```

So a join command could be:

```bash
sudo kubeadm join 10.0.1.10:6443 \
  --token <token> \
  --discovery-token-ca-cert-hash sha256:<hash> \
  --cri-socket unix:///run/containerd/containerd.sock
```

---

# 17. About `--v=5`

You had this in the original guide:

```bash
--v=5
```

This enables more verbose logging.

It is useful when troubleshooting:

```bash
sudo kubeadm join ... --v=5
```

But it is **not required** for a normal worker join.

Therefore, don't include it in the normal command unless you are troubleshooting.

---

# 18. Verify the Cluster

Go back to the control-plane node.

Run:

```bash
kubectl get nodes
```

Expected result:

```text
NAME            STATUS   ROLES           AGE   VERSION
control-plane   Ready    control-plane   ...   v1.34.x
worker-1        Ready    <none>          ...   v1.34.x
worker-2        Ready    <none>          ...   v1.34.x
```

The important part is:

```text
STATUS = Ready
```

---

# 19. Check All Kubernetes Pods

Run:

```bash
kubectl get pods -A
```

You should eventually see the system Pods running.

For example:

```text
kube-system
calico-system
tigera-operator
```

Check more specifically:

```bash
kubectl get pods -n kube-system
```

and:

```bash
kubectl get pods -A
```

---

# 20. Verify Worker Node Container Runtime

On a worker:

```bash
sudo systemctl status containerd
```

You should see:

```text
Active: active (running)
```

Check the kubelet:

```bash
sudo systemctl status kubelet
```

You can also check the node's runtime from the control plane:

```bash
kubectl get nodes -o wide
```

The output contains information such as:

```text
INTERNAL-IP
OS-IMAGE
KERNEL-VERSION
CONTAINER-RUNTIME
```

---

# 21. Useful Verification Commands

## Check nodes

```bash
kubectl get nodes
```

## More detailed node information

```bash
kubectl get nodes -o wide
```

## Check all Pods

```bash
kubectl get pods -A
```

## Check cluster information

```bash
kubectl cluster-info
```

## Check kubelet

```bash
sudo systemctl status kubelet
```

## Check containerd

```bash
sudo systemctl status containerd
```

## Check Kubernetes versions

```bash
kubectl version
```

## Check kubeadm

```bash
kubeadm version
```

---

# 22. Important Kubernetes Components

After the cluster is created, the architecture looks like this:

```text
                     Control Plane
                +-----------------------+
                |                       |
                |   kube-apiserver      |
                |          |            |
                |       etcd             |
                |                       |
                |   scheduler           |
                |   controller-manager  |
                |                       |
                +-----------+-----------+
                            |
                 Kubernetes API :6443
                            |
             +--------------+--------------+
             |                             |
       +-----v-----+                 +-----v-----+
       |  Worker 1 |                 |  Worker 2 |
       +-----------+                 +-----------+
       | kubelet   |                 | kubelet   |
       | containerd|                 | containerd|
       | kube-proxy|                 | kube-proxy|
       +-----------+                 +-----------+
             |                             |
             +-------------+---------------+
                           |
                         Calico
                       Pod Network
```

---

# 23. Why We Need Each Component

### kubeadm

Used to bootstrap the Kubernetes cluster.

```text
kubeadm init
kubeadm join
```

---

### kubelet

Runs on every node.

Its job is to make sure the Pods assigned to that node are actually running.

```text
Control Plane
      |
      v
    kubelet
      |
      v
   Containers
```

---

### kubectl

Your command-line interface for Kubernetes.

For example:

```bash
kubectl get pods
kubectl get nodes
kubectl create deployment nginx --image=nginx
```

---

### containerd

Runs the actual containers.

```text
kubelet
   |
   v
containerd
   |
   v
Container
```

---

### Calico

Provides networking between Kubernetes Pods.

Without a CNI, nodes can be created but Pods cannot communicate correctly.

---

# 24. Common Problems

## Node is NotReady

Check:

```bash
kubectl get nodes
```

Then:

```bash
kubectl get pods -A
```

If Calico is not running, inspect:

```bash
kubectl get pods -A
```

---

## kubelet is failing

Run:

```bash
sudo systemctl status kubelet
```

Then:

```bash
sudo journalctl -u kubelet -xe
```

---

## containerd is failing

Run:

```bash
sudo systemctl status containerd
```

Then:

```bash
sudo journalctl -u containerd -xe
```

---

## Worker cannot join

Check that the worker can reach the control plane:

```bash
nc -vz <control-plane-private-ip> 6443
```

Example:

```bash
nc -vz 10.0.1.10 6443
```

If this fails, check the AWS Security Group and VPC networking.

---

# 25. If the Join Token Expires

The worker join token generated by kubeadm is temporary.

Generate a new command from the control plane:

```bash
kubeadm token create --print-join-command
```

Then use the newly generated command on the worker.

---

# 26. Reset a Failed Cluster

If you are practicing and want to start again on a node:

```bash
sudo kubeadm reset -f
```

Then remove the kubeconfig if necessary:

```bash
rm -rf "$HOME/.kube"
```

Restart the runtime:

```bash
sudo systemctl restart containerd
```

You can then initialize/join the node again.

---

# 27. Final Verification

On the control-plane node:

```bash
kubectl get nodes -o wide
```

Then:

```bash
kubectl get pods -A
```

The desired result is:

```text
All nodes = Ready
```

and Kubernetes system Pods should be running.

---

# 28. Quick Command Summary

## ALL NODES

```bash
sudo swapoff -a

cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward = 1
EOF

sudo sysctl --system
```

Install and configure:

```text
containerd
    ↓
kubelet
kubeadm
kubectl
```

---

## CONTROL PLANE ONLY

```bash
sudo kubeadm init --pod-network-cidr=192.168.0.0/16
```

Configure kubectl:

```bash
mkdir -p "$HOME/.kube"

sudo cp -i /etc/kubernetes/admin.conf "$HOME/.kube/config"

sudo chown "$(id -u)":"$(id -g)" "$HOME/.kube/config"
```

Install Calico:

```bash
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.32.1/manifests/v1_crd_projectcalico_org.yaml

kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.32.1/manifests/tigera-operator.yaml

kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.32.1/manifests/custom-resources.yaml
```

Generate worker join command:

```bash
kubeadm token create --print-join-command
```

---

## WORKER NODES ONLY

If the node was previously initialized:

```bash
sudo kubeadm reset -f
```

Then run the generated command:

```bash
sudo kubeadm join <CONTROL-PLANE-IP>:6443 \
  --token <TOKEN> \
  --discovery-token-ca-cert-hash sha256:<HASH>
```

---

## FINAL CHECK

On the control plane:

```bash
kubectl get nodes
```

```bash
kubectl get pods -A
```

```bash
kubectl cluster-info
```

---

# Official References

* Kubernetes kubeadm installation: https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/
* Creating a kubeadm cluster: https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/
* Kubernetes ports: https://kubernetes.io/docs/reference/networking/ports-and-protocols/
* Container runtimes: https://kubernetes.io/docs/setup/production-environment/container-runtimes/
* Calico Kubernetes installation: https://docs.tigera.io/calico/latest/getting-started/kubernetes/self-managed-onprem/onpremises
