# Kubernetes Workloads Lab

Hands-on practice with Kubernetes workloads using a KIND cluster.

## Environment

KIND cluster:

```text
cws-cluster
├── control-plane
├── worker
└── worker2
```

Kubernetes version:

```text
v1.31.2
```

Nginx images used during practice:

```text
nginx:1.27
nginx:1.26
```

---

# Pod

A Pod is the smallest deployable unit in Kubernetes.

## Manifest

File:

```text
pod.yaml
```

```yaml
apiVersion: v1
kind: Pod

metadata:
  name: nginx-pod
  namespace: nginx-ns

spec:
  containers:
  - name: nginx
    image: nginx:1.27
    ports:
    - containerPort: 80
```

## Create

```bash
kubectl apply -f kind/namespace/namespace.yaml
kubectl apply -f kind/workloads/pod.yaml
```

## Verify

```bash
kubectl get pods -n nginx-ns
```

Detailed information:

```bash
kubectl get pods -n nginx-ns -o wide
```

Describe:

```bash
kubectl describe pod nginx-pod -n nginx-ns
```

## Execute commands inside the Pod

```bash
kubectl exec -it pod/nginx-pod -n nginx-ns -- bash
```

Inside the container:

```bash
curl http://localhost:80
```

The Nginx welcome page was returned successfully.

Exit:

```bash
exit
```

## Delete

```bash
kubectl delete pod nginx-pod -n nginx-ns
```

---

# Deployment

A Deployment manages Pods through a ReplicaSet and maintains the desired number of replicas.

## Manifest

File:

```text
deployment.yaml
```

```yaml
apiVersion: apps/v1
kind: Deployment

metadata:
  name: nginx-depl
  namespace: nginx-ns

spec:
  replicas: 2

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
        image: nginx:1.27
        ports:
        - containerPort: 80
```

## Create

```bash
kubectl apply -f kind/workloads/deployment.yaml
```

## Verify

Deployment:

```bash
kubectl get deployment -n nginx-ns
```

ReplicaSet:

```bash
kubectl get replicaset -n nginx-ns
```

Pods:

```bash
kubectl get pods -n nginx-ns
```

Everything:

```bash
kubectl get all -n nginx-ns
```

The relationship observed during practice:

```text
Deployment
    |
    v
ReplicaSet
    |
    v
Pods
```

With:

```yaml
replicas: 2
```

the Deployment created two Pods.

---

# Scaling

The Deployment was scaled from two replicas to five.

```bash
kubectl scale deployment nginx-depl --replicas=5 -n nginx-ns
```

Verify:

```bash
kubectl get pods -n nginx-ns
```

The Deployment reached:

```text
5/5
```

The Deployment was then scaled back to one replica:

```bash
kubectl scale deployment nginx-depl --replicas=1 -n nginx-ns
```

Verify:

```bash
kubectl get all -n nginx-ns
```

---

# Rolling Update

The Nginx image was updated from:

```text
nginx:1.27
```

to:

```text
nginx:1.26
```

Command:

```bash
kubectl set image deployment/nginx-depl nginx=nginx:1.26 -n nginx-ns
```

Watch the update:

```bash
kubectl get pods -n nginx-ns -w
```

Kubernetes created a new ReplicaSet for the new Pod template.

Observed relationship:

```text
Old ReplicaSet
nginx:1.27
     |
     | scale down
     v
New ReplicaSet
nginx:1.26
```

The old Pod was terminated after the new Pod became ready.

Verify the image:

```bash
kubectl describe pod <pod-name> -n nginx-ns
```

The updated Pod showed:

```text
Image: nginx:1.26
```

---

# Namespace reminder

The workloads are deployed in:

```text
nginx-ns
```

Therefore namespace-specific commands need:

```bash
-n nginx-ns
```

For example:

```bash
kubectl get pods -n nginx-ns
kubectl get deployment -n nginx-ns
kubectl get replicaset -n nginx-ns
```

Without `-n nginx-ns`, kubectl normally checks the `default` namespace.

---

# Commands Practiced

```bash
kubectl apply -f kind/namespace/namespace.yaml

kubectl apply -f kind/workloads/pod.yaml
kubectl get pods -n nginx-ns
kubectl get pods -n nginx-ns -o wide
kubectl describe pod nginx-pod -n nginx-ns
kubectl exec -it pod/nginx-pod -n nginx-ns -- bash
kubectl delete pod nginx-pod -n nginx-ns

kubectl apply -f kind/workloads/deployment.yaml
kubectl get deployment -n nginx-ns
kubectl get replicaset -n nginx-ns
kubectl get pods -n nginx-ns
kubectl get all -n nginx-ns

kubectl scale deployment nginx-depl --replicas=5 -n nginx-ns
kubectl scale deployment nginx-depl --replicas=1 -n nginx-ns

kubectl set image deployment/nginx-depl nginx=nginx:1.26 -n nginx-ns

kubectl get pods -n nginx-ns -w
```

# Concepts Practiced

* Pod
* Deployment
* ReplicaSet created by Deployment
* Multiple Pod replicas
* Scaling
* Rolling updates
* Container image updates
* Namespace-specific resources
* `kubectl get`
* `kubectl describe`
* `kubectl exec`
* `kubectl apply`
* `kubectl delete`
* `kubectl scale`
* `kubectl set image`
