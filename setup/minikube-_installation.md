# Minikube Installation on Ubuntu

This guide installs the following tools:

- Docker
- kubectl
- Minikube

Minikube will use Docker as its driver.

---

## 1. Update packages

Run:

```bash
sudo apt-get update
```

This refreshes the package information on your Ubuntu system.

---

## 2. Install prerequisites

```bash
sudo apt-get install -y ca-certificates curl
```

### Why?

- `ca-certificates` → enables trusted HTTPS connections.
- `curl` → used to download kubectl and Minikube.

---

## 3. Install Docker

```bash
sudo apt-get install -y docker.io
```

Install Docker from the Ubuntu package repository.

Start Docker and configure it to start automatically at boot:

```bash
sudo systemctl enable --now docker
```

### Verify Docker

```bash
sudo docker --version
```

---

## 4. Allow your user to run Docker without sudo

Add your current user to the `docker` group:

```bash
sudo usermod -aG docker "$USER"
```

This allows you to use Docker commands without `sudo`.

For example, after the group change:

```bash
docker ps
```

instead of:

```bash
sudo docker ps
```

### Important

The group change will not immediately apply to your current terminal.

**Log out of Ubuntu, log back in, and then open a NEW terminal.**

Now run:

```bash
docker ps
```

If it works without `sudo`, Docker is configured correctly.

---

## 5. Install kubectl

Set the kubectl version:

```bash
KUBECTL_VERSION="v1.36.0"
```

Download kubectl:

```bash
curl -LO "https://dl.k8s.io/release/${KUBECTL_VERSION}/bin/linux/amd64/kubectl"
```

Make it executable:

```bash
chmod +x kubectl
```

Move it to `/usr/local/bin`:

```bash
sudo mv kubectl /usr/local/bin/kubectl
```

### Verify kubectl

```bash
kubectl version --client
```

---

## 6. Install Minikube

Download the latest Minikube binary:

```bash
curl -LO https://github.com/kubernetes/minikube/releases/latest/download/minikube-linux-amd64
```

Install it:

```bash
sudo install minikube-linux-amd64 /usr/local/bin/minikube
```

Remove the downloaded file:

```bash
rm minikube-linux-amd64
```

### Verify Minikube

```bash
minikube version
```

---

## 7. Start Minikube

Start Minikube using Docker as the driver:

```bash
minikube start --driver=docker
```

Minikube will create a local Kubernetes cluster using Docker.

---

## 8. Verify the Kubernetes cluster

Check the Kubernetes nodes:

```bash
kubectl get nodes
```

You should see a Minikube node with status:

```text
Ready
```

You can also check Minikube:

```bash
minikube status
```

---

# Complete Installation Script

Create a file:

```bash
nano install-minikube.sh
```

Add:

```bash
#!/bin/bash

set -e

echo "===== Updating packages ====="
sudo apt-get update

echo "===== Installing prerequisites ====="
sudo apt-get install -y ca-certificates curl

# --------------------------------------------------
# Docker
# --------------------------------------------------

echo "===== Installing Docker ====="

sudo apt-get install -y docker.io

sudo systemctl enable --now docker

# Add current user to docker group
sudo usermod -aG docker "$USER"

echo "Docker installed."

# --------------------------------------------------
# kubectl
# --------------------------------------------------

echo "===== Installing kubectl ====="

KUBECTL_VERSION="v1.36.0"

curl -LO "https://dl.k8s.io/release/${KUBECTL_VERSION}/bin/linux/amd64/kubectl"

chmod +x kubectl

sudo mv kubectl /usr/local/bin/kubectl

echo "kubectl installed."

# --------------------------------------------------
# Minikube
# --------------------------------------------------

echo "===== Installing Minikube ====="

curl -LO "https://github.com/kubernetes/minikube/releases/latest/download/minikube-linux-amd64"

sudo install minikube-linux-amd64 /usr/local/bin/minikube

rm minikube-linux-amd64

echo "Minikube installed."

# --------------------------------------------------
# Verification
# --------------------------------------------------

echo
echo "===== Versions ====="

docker --version
kubectl version --client
minikube version

echo
echo "===== Setup completed ====="

echo
echo "IMPORTANT:"
echo "Docker group membership has been updated."
echo "Please log out and log back in."
echo "Then open a NEW terminal and run:"
echo
echo "docker ps"
echo
echo "After Docker works, start Minikube with:"
echo "minikube start --driver=docker"
```

Save the file and make it executable:

```bash
chmod +x install-minikube.sh
```

Run it:

```bash
./install-minikube.sh
```

---

# After Installation

After running the script:

### 1. Log out and log back in

Then open a **NEW terminal**.

### 2. Verify Docker

```bash
docker ps
```

### 3. Start Minikube

```bash
minikube start --driver=docker
```

### 4. Verify Kubernetes

```bash
kubectl get nodes
```

### 5. Check Minikube

```bash
minikube status
```

---

# Tools Installed

```text
Ubuntu
  |
  +-- Docker
  |
  +-- kubectl
  |
  +-- Minikube
          |
          +-- Local Kubernetes Cluster
```

### Docker

Runs containers and is used by Minikube as the driver.

### kubectl

Command-line tool used to communicate with and manage Kubernetes.

### Minikube

Creates and manages a local Kubernetes cluster for learning and development.
