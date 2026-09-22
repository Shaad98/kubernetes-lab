# Kind + NGINX Ingress Lab — Complete Commands & Practical Notes

This document records the practical command sequence, configuration, debugging, and lessons from the Kind + NGINX Ingress lab.

## Lab architecture

```text
Browser
   │
   │ HTTP / HTTPS
   ▼
Host machine
   │
   │ host port mapping
   ▼
Kind Docker node
   │
   ▼
NGINX Ingress Controller
   │
   │ Ingress rules
   ├── /api → backend-service:8080
   └── /    → nginx-service:80
                 │
                 ├── backend Pods
                 └── NGINX Pods

Backend Pods
   │
   ▼
postgres-service:5432
   │
   ▼
PostgreSQL Pod
   │
   ▼
PersistentVolumeClaim → PersistentVolume
```

---

# 1. Application setup commands

Check namespaces:

```bash
kubectl get ns
```

Check all resources in the application namespace:

```bash
kubectl get all -n nginx-ns
```

Apply the Secret:

```bash
kubectl apply -f secret.yaml
```

Apply the ConfigMap:

```bash
kubectl apply -f config-map.yaml
```

Apply the PersistentVolume:

```bash
kubectl apply -f pv.yaml
```

Apply the PersistentVolumeClaim:

```bash
kubectl apply -f pvc.yaml
```

Check storage:

```bash
kubectl get pv
kubectl get pvc -n nginx-ns
```

> `PersistentVolume` is cluster-scoped, so the namespace is not relevant for `kubectl get pv`.

Apply NGINX:

```bash
kubectl apply -f nginx-deployment.yaml
kubectl apply -f nginx-service.yaml
```

Apply PostgreSQL:

```bash
kubectl apply -f postgres-deployment.yaml
kubectl apply -f postgres-service.yaml
```

Apply Spring Boot backend:

```bash
kubectl apply -f backend-deployment.yaml
kubectl apply -f backend-service.yaml
```

Check the application:

```bash
kubectl get all -n nginx-ns
kubectl get pods -n nginx-ns -o wide
```

---

# 2. Backend image and environment variables

The custom Spring Boot image is:

```text
shaad98/k8s-user-management:vs2
```

The container listens on port:

```text
8080
```

The Deployment receives three environment variables.

## DB_URL

```yaml
- name: DB_URL
  valueFrom:
    configMapKeyRef:
      name: application-data
      key: DB_URL
```

Value:

```text
jdbc:postgresql://postgres-service:5432/workdb
```

## DB_USERNAME

```yaml
- name: DB_USERNAME
  valueFrom:
    configMapKeyRef:
      name: application-data
      key: POSTGRES_USER
```

Value:

```text
postgres
```

## DB_PASSWORD

```yaml
- name: DB_PASSWORD
  valueFrom:
    secretKeyRef:
      name: postgres-secret
      key: POSTGRES_PASSWORD
```

So the configuration flow is:

```text
DB_URL
  ↓
ConfigMap / application-data / DB_URL

DB_USERNAME
  ↓
ConfigMap / application-data / POSTGRES_USER

DB_PASSWORD
  ↓
Secret / postgres-secret / POSTGRES_PASSWORD
```

---

# 3. ConfigMap

Current configuration:

```yaml
kind: ConfigMap
apiVersion: v1

metadata:
  name: application-data
  namespace: nginx-ns

data:
  DB_URL: "jdbc:postgresql://postgres-service:5432/workdb"
  POSTGRES_USER: postgres
  POSTGRES_DB: workdb
```

Check it:

```bash
kubectl get configmap -n nginx-ns
```

---

# 4. Secret

Current Secret shape:

```yaml
kind: Secret
apiVersion: v1

metadata:
  name: postgres-secret
  namespace: nginx-ns

data:
  POSTGRES_PASSWORD: cm9vdDEyMw==
```

Check it:

```bash
kubectl get secret -n nginx-ns
```

Remember that Base64 is encoding, not encryption.

Do not treat a Git-committed Base64 value as a secure production secret.

---

# 5. PostgreSQL and storage

PostgreSQL is reached through:

```text
postgres-service:5432
```

The backend JDBC URL is:

```text
jdbc:postgresql://postgres-service:5432/workdb
```

Storage relationship:

```text
PostgreSQL Pod
      ↓
PersistentVolumeClaim
      ↓
PersistentVolume
      ↓
/mnt/data on the Kind node/hostPath
```

The PVC requests 1 GiB and uses the `local-storage` StorageClass.

---

# 6. Create the Ingress resource

Edit the Ingress manifest:

```bash
vim ingress.yaml
```

Current routing:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress

metadata:
  name: app-ingress
  namespace: nginx-ns

spec:
  ingressClassName: nginx

  rules:
    - http:
        paths:
          - path: /api
            pathType: Prefix
            backend:
              service:
                name: backend-service
                port:
                  number: 8080

          - path: /
            pathType: Prefix
            backend:
              service:
                name: nginx-service
                port:
                  number: 80
```

Apply it:

```bash
kubectl apply -f ingress.yaml
```

Check it:

```bash
kubectl get ingress -n nginx-ns
kubectl get ingress -n nginx-ns -o yaml
```

The routes are:

```text
/           → nginx-service:80
/api        → backend-service:8080
/api/users  → backend-service:8080
```

---

# 7. Why the Ingress Controller is required

An `Ingress` resource is a routing definition. It does not itself receive and
process HTTP traffic.

The NGINX Ingress Controller is the software that watches the Ingress resources
and handles the actual request routing.

Conceptually:

```text
Ingress resource
      ↓
routing rules
      ↓
NGINX Ingress Controller
      ↓
Kubernetes Service
      ↓
Pod
```

Install ingress-nginx with:

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.15.1/deploy/static/provider/cloud/deploy.yaml
```

Check the controller:

```bash
kubectl get pods -n ingress-nginx
kubectl get pods -n ingress-nginx -o wide
```

---

# 8. Kind runs Kubernetes nodes as Docker containers

Kind creates the Kubernetes nodes as Docker containers.

Conceptually:

```text
Ubuntu host
   │
   └── Docker
       ├── cws-cluster-control-plane
       ├── cws-cluster-worker
       └── cws-cluster-worker2
```

Therefore a Kubernetes node port is not automatically the same as a port on the
Ubuntu host.

For example, if the ingress Service shows:

```text
80:30433/TCP
```

that means:

```text
Service port = 80
NodePort     = 30433
```

It does not by itself guarantee:

```bash
curl http://localhost:30433/api
```

will work on the host.

The Kind Docker node must also be exposed to the host.

---

# 9. Kind cluster configuration

The cluster configuration used in this lab is:

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
  extraPortMappings:
  - containerPort: 80
    hostPort: 80
    protocol: TCP
  - containerPort: 443
    hostPort: 443
    protocol: TCP
```

The `extraPortMappings` are attached to the third node.

Conceptually:

```text
Ubuntu host :80
      ↓
Kind Docker node :80
```

and:

```text
Ubuntu host :443
      ↓
Kind Docker node :443
```

---

# 10. Finding the node where the Ingress Controller runs

Use:

```bash
kubectl get pods -n ingress-nginx -o wide
```

The `NODE` column is the important part.

Example:

```text
NAME                                      READY   STATUS    IP           NODE
ingress-nginx-controller-xxxxx            1/1     Running   10.244.2.14  cws-cluster-worker2
```

This tells you that the controller Pod is running on:

```text
cws-cluster-worker2
```

Always check with `-o wide` instead of assuming Kubernetes put the Pod on a
specific worker.

---

# 11. `extraPortMappings` vs `hostPort`

These are different mechanisms and should not be treated as the same thing.

## `extraPortMappings`

Configured in the Kind cluster configuration:

```yaml
extraPortMappings:
- containerPort: 80
  hostPort: 80
```

Its purpose is to expose a Kind Docker node port to the real host.

Conceptually:

```text
Host machine
    ↓
Kind Docker node
```

## `hostPort`

Configured in a Kubernetes Pod/Deployment:

```yaml
ports:
- name: http
  containerPort: 80
  hostPort: 80
  protocol: TCP
```

Its purpose is to bind that container port to the node where the Pod is running.

Conceptually:

```text
Kubernetes node :80
        ↓
Ingress Controller Pod :80
```

So the two layers can be viewed as:

```text
Host
  ↓
extraPortMappings
  ↓
Kind node
  ↓
hostPort
  ↓
Pod
```

---

# 12. Editing the Ingress Controller Deployment

A live Deployment can be edited directly with:

```bash
kubectl edit deployment ingress-nginx-controller -n ingress-nginx
```

This opens the live Deployment in the configured editor.

Find the controller container:

```yaml
spec:
  template:
    spec:
      containers:
      - name: controller
```

Inside that container, update the ports section with the required `hostPort`.
For example:

```yaml
ports:
- name: http
  containerPort: 80
  hostPort: 80
  protocol: TCP

- name: https
  containerPort: 443
  hostPort: 443
  protocol: TCP
```

Save and exit the editor.

Kubernetes will update the Deployment and create/update the controller Pod.

Then verify:

```bash
kubectl get deployment ingress-nginx-controller -n ingress-nginx
kubectl get pods -n ingress-nginx -o wide
```

Also inspect the full namespace:

```bash
kubectl get all -n ingress-nginx -o wide
```

### Important warning about `kubectl edit`

`kubectl edit` changes the live resource. It does not modify the original
upstream YAML downloaded from GitHub.

If the controller is later re-applied from the upstream manifest, the manual
change can be lost.

For a reusable Git repository, a patch or managed manifest is better than
relying only on a manual `kubectl edit`.

---

# 13. Why the controller node matters when using `hostPort`

When a Pod specifies:

```yaml
hostPort: 80
```

the binding belongs to the node on which that Pod is running.

Therefore:

```bash
kubectl get pods -n ingress-nginx -o wide
```

must be used to see the actual node.

If the lab is designed around a particular Kind node being exposed to the host,
you need the controller Pod to be placed on the intended node.

Kubernetes does not guarantee a particular node just because the Pod was
recreated.

For deterministic placement, use scheduling controls such as:

```text
nodeSelector
node affinity
```

Deleting a controller Pod only causes the Deployment to create a replacement;
it does not guarantee that the new Pod will move to another node.

---

# 14. Deleting the Ingress Controller Pod

A Pod can be deleted to force the Deployment to recreate it:

```bash
kubectl delete pod/ingress-nginx-controller-7c87c8df67-88vrh -n ingress-nginx
```

Then check:

```bash
kubectl get pods -n ingress-nginx -o wide
```

Another Pod can similarly be deleted:

```bash
kubectl delete pod/ingress-nginx-controller-64547f59c8-h4m8c -n ingress-nginx
```

Again, deletion does not guarantee rescheduling onto a different node.

---

# 15. Inspect all Ingress Controller resources

```bash
kubectl get all -n ingress-nginx

# Port-forward Ingress Controller for local testing
sudo -E kubectl port-forward -n ingress-nginx service/ingress-nginx-controller 81:80
```

With node/IP information:

```bash
kubectl get all -n ingress-nginx -o wide
```

The most useful checks are:

```bash
kubectl get pods -n ingress-nginx -o wide
kubectl get svc -n ingress-nginx
kubectl get deployment -n ingress-nginx
```

---

# 16. Ingress Controller Service and NodePort

Check:

```bash
kubectl get svc -n ingress-nginx
```

Example:

```text
ingress-nginx-controller   LoadBalancer   ...   80:30433/TCP,443:31450/TCP
```

Interpretation:

```text
HTTP:
Service port = 80
NodePort     = 30433

HTTPS:
Service port = 443
NodePort     = 31450
```

A `LoadBalancer` Service can remain with:

```text
EXTERNAL-IP = <pending>
```

on a local Kind cluster because there is no cloud load balancer automatically
providing the external address.

---

# 17. Test application Services from inside the cluster

Before troubleshooting host/Ingress networking, prove that the Services and
applications work inside the cluster.

Create a temporary curl Pod:

```bash
kubectl run curl-test --rm -it --image=curlimages/curl -n nginx-ns -- sh
```

Inside the temporary Pod, test NGINX:

```bash
curl -i http://nginx-service:80
```

Test Spring Boot:

```bash
curl -i http://backend-service:8080/api
```

Test users endpoint:

```bash
curl -i http://backend-service:8080/api/users
```

If these work, then:

```text
Service → Pod → Application
```

is already working.

At that point, investigate the Ingress/controller/host networking layer instead
of changing the application or Service unnecessarily.

---

# 18. Check backend logs

First find the backend Pods:

```bash
kubectl get pods -n nginx-ns -o wide
```

Then:

```bash
kubectl logs pod/<backend-pod-name> -n nginx-ns
```

Example:

```bash
kubectl logs pod/backend-depl-6bc7bc4d94-25kjr -n nginx-ns
```

A common mistake is forgetting the namespace:

```bash
kubectl logs pod/backend-depl-6bc7bc4d94-25kjr
```

If the Pod is in `nginx-ns`, include:

```bash
-n nginx-ns
```

---

# 19. Test and inspect the Ingress

Edit:

```bash
vim ingress.yaml
```

Apply:

```bash
kubectl apply -f ingress.yaml
```

Check:

```bash
kubectl get ingress -n nginx-ns
kubectl get ingress -n nginx-ns -o yaml
```

Delete and recreate if needed:

```bash
kubectl delete -f ingress.yaml
kubectl apply -f ingress.yaml
```

Describe it for debugging:

```bash
kubectl describe ingress app-ingress -n nginx-ns
```

---

# 20. Why `/nginx` gave 404

An earlier route used:

```yaml
- path: /nginx
  pathType: Prefix
  backend:
    service:
      name: nginx-service
      port:
        number: 80
```

This does not create an `/nginx` resource inside the NGINX server.

The request flow is:

```text
GET /nginx
    ↓
Ingress
    ↓
nginx-service
    ↓
NGINX receives /nginx
```

The stock NGINX image normally serves its welcome page at:

```text
/
```

not:

```text
/nginx
```

Therefore:

```text
GET /       → default NGINX welcome page
GET /nginx  → usually 404 with stock NGINX configuration
```

For this lab, using `/` for the NGINX route is simpler:

```yaml
- path: /
  pathType: Prefix
  backend:
    service:
      name: nginx-service
      port:
        number: 80
```

---

# 21. Non-standard host ports and redirects

Using a custom host port such as:

```text
81
```

or:

```text
82
```

can work for simple requests.

However, applications may generate redirects or absolute URLs.

Example:

```text
Client
  ↓
http://example:81
  ↓
Application
  ↓
HTTP redirect
  ↓
new URL
```

Whether the redirect keeps the expected external port depends on the application
and proxy/forwarded-header configuration.

If the application thinks the externally visible URL is one thing while the
browser actually reached it through another port, the application can generate
a redirect to an unexpected or unreachable endpoint.

This can look like:

```text
Request works
   ↓
Application redirects
   ↓
Browser follows redirect
   ↓
Destination is unreachable
```

That does not necessarily mean the application process crashed.

Check:

```bash
kubectl get pods -n nginx-ns
kubectl logs -n nginx-ns <backend-pod-name>
```

For a simple HTTP/HTTPS lab, prefer the standard external ports when possible:

```text
HTTP  → 80
HTTPS → 443
```

This keeps redirects, generated links, browser behavior, and proxy configuration
easier to reason about.

---

# 22. `curl http://api/users` and host-side testing

A host-side request such as:

```bash
curl http://api/users
```

depends on the host being able to resolve `api` and on the host networking being
correctly configured.

If DNS, `/etc/hosts`, ports, or the Kind-to-host mapping are not configured,
the request can fail before Kubernetes ever receives it.

For Kubernetes routing tests, first prove the internal Service path:

```bash
curl -i http://backend-service:8080/api/users
```

Then test through the Ingress once host networking is known to work.

---

# 23. Recommended debugging order

When something fails, debug from the inside outward:

```text
1. Application Pod
       ↓
2. Kubernetes Service
       ↓
3. Ingress resource
       ↓
4. Ingress Controller
       ↓
5. Kind node networking
       ↓
6. Host machine
       ↓
7. Browser / external client
```

Useful commands:

```bash
kubectl get pods -n nginx-ns -o wide
kubectl get svc -n nginx-ns
kubectl get ingress -n nginx-ns
kubectl get pods -n ingress-nginx -o wide
kubectl get svc -n ingress-nginx
kubectl logs -n ingress-nginx deployment/ingress-nginx-controller
```

---

# 24. Complete command list from this lab

The original shell history used during this lab was:

```bash
vim backend-service.yaml
kubectl get ns
kubectl get all -n nginx-ns
kubectl apply -f secret.yaml
kubectl apply -f config-map.yaml
kubectl apply -f pv.yaml
kubectl apply -f pvc.yaml
kubectl get pv -n nginx-ns
kubectl get pvc -n nginx-ns
kubectl apply -f nginx-deployment.yaml
kubectl apply -f nginx-service.yaml
kubectl apply -f postgres-deployment.yaml
kubectl apply -f postgres-service.yaml
kubectl apply -f backend-deployment.yaml
kubectl apply -f backend-service.yaml
vim ingress.yaml
kubectl apply -f ingress.yaml
kubectl get pods -o wide
kubectl get pods -n ingress-nginx -o wide
kubectl get ns
kubectl delete -f ingress.yaml
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.15.1/deploy/static/provider/cloud/deploy.yaml
vim ingress.yaml
kubectl get pods -n ingress-nginx -o wide
kubectl delete pod/ingress-nginx-controller-7c87c8df67-88vrh -n ingress-nginx
kubectl get pods -n ingress-nginx -o wide
kubectl get all -n ingress-nginx -o wide
kubectl edit deployment ingress-nginx-controller -n ingress-nginx
kubectl get all -n ingress-nginx -o wide
kubectl get all -n ingress-nginx
kubectl get pods -n ingress-nginx -o wide
kubectl delete pod/ingress-nginx-controller-64547f59c8-h4m8c -n ingress-nginx
kubectl get pods -n ingress-nginx -o wide
vim ingress.yaml
curl http://api/users
kubectl get all -n nginx-ns
kubectl logs pod/backend-depl-6bc7bc4d94-25kjr
kubectl logs pod/backend-depl-6bc7bc4d94-25kjr -n nginx-ns
kubectl get all -n nginx-ns
kubectl get all -n ingress-nginx
cat backend-service.yaml
cat nginx-service.yaml
cat postgres-service.yaml
cat backend-deployment.yaml
kubectl get ingress -n nginx-ns -o yaml
kubectl apply -f ingress.yaml
kubectl get ingress -n nginx-ns -o yaml
kubectl get all -n nginx-ns
cat nginx-deployment.yaml
cat nginx-service.yaml
vim ingress.yaml
kubectl apply -f ingress.yaml
clear
cat *
cd ..
cd K8s_Labs/
git status
ls
cd kubernetes-lab/
git status
tree .
history
```

---

# 25. Most important commands to remember

### Install ingress-nginx

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.15.1/deploy/static/provider/cloud/deploy.yaml
```

### Find the controller node

```bash
kubectl get pods -n ingress-nginx -o wide
```

### Edit the controller Deployment

```bash
kubectl edit deployment ingress-nginx-controller -n ingress-nginx
```

### Check controller resources

```bash
kubectl get all -n ingress-nginx -o wide
```

### Check controller Service / NodePort

```bash
kubectl get svc -n ingress-nginx
```

### Check application

```bash
kubectl get all -n nginx-ns
```

### Test Services internally

```bash
kubectl run curl-test --rm -it --image=curlimages/curl -n nginx-ns -- sh
```

Then:

```bash
curl -i http://nginx-service:80
curl -i http://backend-service:8080/api
curl -i http://backend-service:8080/api/users
```

---

# 26. Final mental model

```text
Ingress resource
      │
      │ says where requests should go
      ▼
NGINX Ingress Controller
      │
      │ actually handles HTTP/HTTPS routing
      ▼
Kubernetes Service
      │
      │ stable endpoint
      ▼
Pod
```

For the Kind host networking layer:

```text
Ubuntu Host
      │
      │ extraPortMappings
      ▼
Kind Docker Node
      │
      │ hostPort (when configured)
      ▼
Ingress Controller Pod
```

The key distinction is:

```text
extraPortMappings = host → Kind Docker node
hostPort           = node → Pod
Ingress            = routing rules
Ingress Controller = software implementing those rules
Service            = stable endpoint for Pods
```

And the practical rule learned from this lab is:

```text
Test from inside the cluster first.
Then debug Ingress.
Then debug Kind node networking.
Then debug the host/browser path.
```