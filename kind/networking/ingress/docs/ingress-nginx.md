# Kind + NGINX Ingress Networking Lab

This document contains the practical setup, configuration, debugging steps, and lessons learned while running **NGINX Ingress on a Kubernetes cluster created with Kind**.

The purpose of this lab is not only to make Ingress work, but to understand what is happening between the **host machine, Docker/Kind nodes, the Ingress Controller, Kubernetes Services, and application Pods**.

---

## 1. Final traffic flow

The complete flow in this lab is:

```text
Browser
   │
   │ HTTP / HTTPS
   ▼
Ubuntu Host
   │
   │ host port 80 / 443
   ▼
Kind Docker Node
   │
   │ node networking / hostPort
   ▼
NGINX Ingress Controller Pod
   │
   │ Ingress routing rules
   ▼
Kubernetes Service
   │
   ▼
Application Pod(s)
```

For this lab there are two application routes:

```text
/           → nginx-service:80
/api        → backend-service:8080
```

And the backend communicates with PostgreSQL through:

```text
backend Pod
    ↓
postgres-service:5432
    ↓
PostgreSQL Pod
```

---

# 2. Install the NGINX Ingress Controller

Kubernetes `Ingress` objects are only routing definitions. They need an **Ingress Controller** to actually process HTTP/HTTPS traffic.

This lab uses **ingress-nginx**.

Install it with:

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.15.1/deploy/static/provider/cloud/deploy.yaml
```

## Why is this required?

Creating this:

```yaml
kind: Ingress
```

does **not** create the software that receives requests.

The roles are different:

```text
Ingress Resource
    │
    │ contains rules
    ▼
Ingress Controller
    │
    │ actually receives and routes requests
    ▼
Kubernetes Service
```

The installation manifest creates the resources needed by ingress-nginx, including the controller Deployment and its Service.

## When is this needed?

Use an Ingress Controller when you want one HTTP/HTTPS entry point to route traffic to different Kubernetes Services based on paths or hosts.

Example:

```text
example.com/       → frontend-service
example.com/api    → backend-service
```

---

# 3. Verify the Ingress Controller

Check the controller Pod:

```bash
kubectl get pods -n ingress-nginx
```

For networking, the most important command is:

```bash
kubectl get pods -n ingress-nginx -o wide
```

The `-o wide` output shows the **NODE** where the controller Pod is running.

Example:

```text
NAME                                      READY   STATUS    IP           NODE
ingress-nginx-controller-xxxxx            1/1     Running   10.244.2.14  cws-cluster-worker2
```

This tells us:

```text
Ingress Controller Pod
        ↓
cws-cluster-worker2
```

This becomes important when using `hostPort`.

---

# 4. Kind nodes are Docker containers

Kind does not normally create full virtual machines for its nodes. The Kubernetes nodes are Docker containers.

Conceptually:

```text
Ubuntu Host
    │
    ▼
Docker
    ├── cws-cluster-control-plane
    ├── cws-cluster-worker
    └── cws-cluster-worker2
```

Therefore, a Kubernetes node port is not automatically the same as a port on the Ubuntu host.

For example, if Kubernetes shows:

```text
30433
```

that does not automatically mean this will work from the host:

```bash
curl http://localhost:30433
```

The Kind Docker node must also be exposed to the host.

This is one of the main networking differences to remember when using Kind.

---

# 5. Kind cluster configuration

The lab cluster is created with:

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

Create the cluster with:

```bash
kind create cluster --config cluster.yaml
```

The important part is:

```yaml
extraPortMappings:
- containerPort: 80
  hostPort: 80

- containerPort: 443
  hostPort: 443
```

This maps the host ports to the **Kind Docker node**.

Conceptually:

```text
Ubuntu Host :80
      │
      ▼
Kind Docker Node :80
```

and:

```text
Ubuntu Host :443
      │
      ▼
Kind Docker Node :443
```

---

# 6. Important: `extraPortMappings` does not choose where a Pod runs

This is a very important concept.

The following configuration:

```yaml
extraPortMappings:
- containerPort: 80
  hostPort: 80
```

does **not** tell Kubernetes:

> Run my Ingress Controller Pod on this node.

It only creates a host-to-Kind-node port mapping.

Kubernetes scheduling is a separate process.

Check the node where the controller actually runs:

```bash
kubectl get pods -n ingress-nginx -o wide
```

Therefore, in a Kind lab using `hostPort`, keep these two things aligned:

```text
Host port mapping
        ↓
Kind node receiving host traffic
        ↓
Ingress Controller Pod with hostPort
        ↓
Pod must be on that node
```

---

# 7. `extraPortMappings` vs `hostPort`

These are often confused, but they solve different parts of the problem.

## `extraPortMappings`

This belongs to the **Kind cluster configuration**.

Example:

```yaml
extraPortMappings:
- containerPort: 80
  hostPort: 80
```

Purpose:

```text
Host machine
    ↓
Kind Docker node
```

It is configured when the Kind cluster is created.

---

## `hostPort`

This belongs to the **Pod/container configuration**.

Example:

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

Purpose:

```text
Kubernetes node
    ↓
Ingress Controller container
```

The `hostPort` is associated with the node where that Pod is actually running.

---

# 8. Running Ingress without `kubectl port-forward`

`kubectl port-forward` is useful for development and debugging, but it is a temporary forwarding mechanism.

For a Kind lab, another setup is to use the ingress controller's `hostPort`.

The controller's container ports can be configured like:

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

Then the intended traffic path is:

```text
Browser
   ↓
Ubuntu Host :80
   ↓
Kind node :80
   ↓
Ingress Controller Pod :80
```

And for HTTPS:

```text
Browser
   ↓
Ubuntu Host :443
   ↓
Kind node :443
   ↓
Ingress Controller Pod :443
```

### Important

When using this approach, the node receiving the Kind `extraPortMappings` must line up with the node on which the ingress controller Pod is running.

If the mapping is on `cws-cluster-worker2` but the ingress controller Pod is on `cws-cluster-worker`, a direct `localhost:80` path will not behave as expected when the `hostPort` is bound only on the controller's Pod node.

---

# 9. Finding the node where Ingress Controller runs

Use:

```bash
kubectl get pods -n ingress-nginx -o wide
```

Example:

```text
ingress-nginx-controller-64547f59c8-7s8cz   1/1   Running   10.244.2.14   cws-cluster-worker2
```

The last column is the node.

You can also inspect the nodes themselves:

```bash
kubectl get nodes -o wide
```

---

# 10. What if the controller is on the wrong node?

For a learning lab, you can delete the controller Pod and let its Deployment recreate it:

```bash
kubectl delete pod <ingress-controller-pod> -n ingress-nginx
```

Then check:

```bash
kubectl get pods -n ingress-nginx -o wide
```

Repeat only when necessary.

## Important scheduler detail

Deleting a Pod does **not guarantee** that the replacement will be scheduled on a different node.

Kubernetes chooses a suitable node according to the scheduler rules.

If you need deterministic placement, use a scheduling rule such as a `nodeSelector` or node affinity.

For example, conceptually:

```yaml
nodeSelector:
  kubernetes.io/hostname: cws-cluster-worker2
```

The exact scheduling configuration depends on the node labels and the final ingress-nginx Deployment configuration.

The important lesson is:

> `hostPort` is bound on the node where the Pod is running.

---

# 11. Ingress Controller Service

After installing ingress-nginx, check its Service:

```bash
kubectl get svc -n ingress-nginx
```

A Kind setup may show something similar to:

```text
NAME                                 TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)
ingress-nginx-controller             LoadBalancer   10.96.x.x       <pending>     80:30433/TCP,443:31450/TCP
ingress-nginx-controller-admission   ClusterIP      10.96.x.x       <none>        443/TCP
```

For example:

```text
80:30433/TCP
```

means:

```text
Service port = 80
NodePort     = 30433
```

And:

```text
443:31450/TCP
```

means:

```text
Service port = 443
NodePort     = 31450
```

In a local Kind cluster, a `LoadBalancer` Service can remain:

```text
EXTERNAL-IP = <pending>
```

because there is no cloud provider automatically assigning an external load balancer IP.

That alone does not mean ingress-nginx is broken.

---

# 12. Why a NodePort may not work on `localhost`

Suppose the Service says:

```text
80:30433/TCP
```

You might try:

```bash
curl http://localhost:30433/api
```

and get:

```text
curl: (7) Failed to connect to localhost port 30433
```

That can happen because the NodePort exists in the Kubernetes/Kind node networking environment, while the Ubuntu host is outside that network namespace.

The important distinction is:

```text
Kubernetes NodePort
        ≠
Ubuntu host localhost automatically
```

An explicit host-to-Kind-node mapping is needed for the host to reach that port directly.

---

# 13. Application namespace

The application resources are placed in:

```text
nginx-ns
```

Namespace definition:

```yaml
kind: Namespace
apiVersion: v1

metadata:
  name: nginx-ns
```

This keeps the demo application resources grouped together.

Check it with:

```bash
kubectl get ns
```

---

# 14. Backend Deployment

The backend Deployment runs two replicas:

```yaml
kind: Deployment
apiVersion: apps/v1

metadata:
  name: backend-depl
  namespace: nginx-ns

spec:
  replicas: 2
  selector:
    matchLabels:
      app: backend
  template:
    metadata:
      labels:
        app: backend
    spec:
      containers:
      - name: backend
        image: shaad98/k8s-user-management:vs2
        ports:
        - containerPort: 8080
```

The custom image is:

```text
shaad98/k8s-user-management:vs2
```

The application listens on port:

```text
8080
```

Two replicas mean Kubernetes creates two backend Pods.

---

# 15. Backend environment variables

The backend receives three environment variables:

```text
DB_URL
DB_PASSWORD
DB_USERNAME
```

They come from Kubernetes ConfigMap/Secret resources.

## `DB_URL`

```yaml
- name: DB_URL
  valueFrom:
    configMapKeyRef:
      name: application-data
      key: DB_URL
```

The value is:

```text
jdbc:postgresql://postgres-service:5432/workdb
```

So the backend connects to the Kubernetes Service named:

```text
postgres-service
```

not directly to a PostgreSQL Pod IP.

---

## `DB_USERNAME`

```yaml
- name: DB_USERNAME
  valueFrom:
    configMapKeyRef:
      name: application-data
      key: POSTGRES_USER
```

The value comes from the ConfigMap:

```text
POSTGRES_USER=postgres
```

Therefore:

```text
DB_USERNAME=postgres
```

---

## `DB_PASSWORD`

```yaml
- name: DB_PASSWORD
  valueFrom:
    secretKeyRef:
      name: postgres-secret
      key: POSTGRES_PASSWORD
```

The Secret currently contains:

```yaml
POSTGRES_PASSWORD: cm9vdDEyMw==
```

which is the Base64 representation of:

```text
root123
```

So, for this lab, the three environment values are effectively:

```text
DB_URL=jdbc:postgresql://postgres-service:5432/workdb
DB_USERNAME=postgres
DB_PASSWORD=root123
```

> This is a lab configuration. Base64 is encoding, not encryption. Do not treat a Git-tracked Kubernetes Secret as secure production secret management.

---

# 16. ConfigMap

The ConfigMap is:

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

It stores non-sensitive application configuration.

The backend uses:

```text
DB_URL
POSTGRES_USER
```

PostgreSQL uses:

```text
POSTGRES_USER
POSTGRES_DB
```

---

# 17. Backend Service

The backend Service is:

```yaml
kind: Service
apiVersion: v1

metadata:
  name: backend-service
  namespace: nginx-ns

spec:
  selector:
    app: backend
  ports:
  - protocol: TCP
    port: 8080
    targetPort: 8080
  type: ClusterIP
```

The important mapping is:

```text
backend-service:8080
        ↓
backend Pod:8080
```

The Ingress sends `/api` traffic to this Service.

---

# 18. NGINX Deployment

The NGINX Deployment has three replicas:

```yaml
kind: Deployment
apiVersion: apps/v1

metadata:
  name: nginx-depl
  namespace: nginx-ns

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
        image: nginx:1.27
        ports:
        - containerPort: 80
```

So:

```text
3 NGINX Pods
```

are managed by the Deployment.

---

# 19. NGINX Service

The NGINX Service is:

```yaml
kind: Service
apiVersion: v1

metadata:
  name: nginx-service
  namespace: nginx-ns

spec:
  selector:
    app: nginx
  ports:
  - port: 80
    targetPort: 80
    protocol: TCP
  type: ClusterIP
```

The Service gives a stable name:

```text
nginx-service:80
```

and sends traffic to the NGINX Pods on port `80`.

---

# 20. PostgreSQL Deployment

The PostgreSQL Deployment has one replica:

```yaml
replicas: 1
```

It uses the ConfigMap and Secret for its environment variables and a PersistentVolumeClaim for storage.

The container should conceptually be reachable on PostgreSQL's normal port:

```text
5432
```

The current Service uses `5432` as the target port.

## Small configuration cleanup

The current PostgreSQL Deployment declares:

```yaml
ports:
- containerPort: 80
```

while PostgreSQL normally listens on `5432`.

`containerPort` is mostly descriptive metadata and does not make the process listen on that port. Since the Service already targets `5432`, the application can still work.

For clarity, the Deployment declaration should preferably be:

```yaml
ports:
- containerPort: 5432
```

---

# 21. PostgreSQL Service

The PostgreSQL Service is:

```yaml
kind: Service
apiVersion: v1

metadata:
  name: postgres-service
  namespace: nginx-ns

spec:
  selector:
    app: postgres
  ports:
  - protocol: TCP
    port: 5432
    targetPort: 5432
  type: ClusterIP
```

So the backend connects using:

```text
postgres-service:5432
```

This is one of the most useful Kubernetes concepts in the lab:

```text
Pod IPs can change
        ↓
Service name stays stable
```

---

# 22. PersistentVolume and PersistentVolumeClaim

The PVC:

```yaml
kind: PersistentVolumeClaim
apiVersion: v1

metadata:
  name: local-pvc
  namespace: nginx-ns

spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
  storageClassName: local-storage
```

The PersistentVolume:

```yaml
kind: PersistentVolume
apiVersion: v1

metadata:
  name: local-pv

spec:
  capacity:
    storage: 1Gi
  accessModes:
  - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: local-storage
  hostPath:
    path: /mnt/data
```

The relationship is:

```text
PostgreSQL Pod
      ↓
PersistentVolumeClaim
      ↓
PersistentVolume
      ↓
hostPath /mnt/data
```

This is useful for learning storage concepts. `hostPath` is not a general production-grade database storage design.

---

# 23. Ingress resource

The Ingress used by the application is:

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

The routing is:

```text
/              → nginx-service:80
/api           → backend-service:8080
/api/users     → backend-service:8080
/api/anything  → backend-service:8080
```

Because `/api` uses:

```yaml
pathType: Prefix
```

all matching paths beginning with `/api` are sent to the backend Service.

---

# 24. Important lesson: `/nginx` does not automatically mean the NGINX welcome page

A previous configuration used:

```yaml
- path: /nginx
```

for the NGINX Service.

That does not create an `/nginx` resource inside NGINX.

The default NGINX image serves its welcome page at:

```text
/
```

not:

```text
/nginx
```

So the request flow was:

```text
GET /nginx
    ↓
Ingress
    ↓
nginx-service
    ↓
NGINX
    ↓
NGINX receives /nginx
```

The stock NGINX configuration normally returns `404` for `/nginx`.

Therefore:

```text
GET /      → NGINX welcome page
GET /nginx → 404 with the default NGINX configuration
```

The clean lab route is:

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

# 25. Test the application from inside the cluster first

Before debugging the external network path, test the internal Kubernetes network.

Start a temporary curl Pod:

```bash
kubectl run curl-test \
  --rm -it \
  --image=curlimages/curl \
  -n nginx-ns \
  -- sh
```

Then test NGINX:

```bash
curl -i http://nginx-service:80
```

Expected result:

```text
HTTP/1.1 200 OK
```

Test Spring Boot:

```bash
curl -i http://backend-service:8080/api
```

Test another backend endpoint:

```bash
curl -i http://backend-service:8080/api/users
```

If these work, then the internal path is already proven:

```text
Service → Pod → Application
```

Do not change a working application Service just because external access is failing.

---

# 26. Debugging order

Use this order when troubleshooting:

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
6. Ubuntu host
       ↓
7. Browser
```

This prevents mixing separate networking problems together.

---

# 27. Useful commands

## Cluster

```bash
kind get clusters
docker ps
```

## Nodes

```bash
kubectl get nodes -o wide
```

## Application Pods

```bash
kubectl get pods -n nginx-ns -o wide
```

## Services

```bash
kubectl get svc -n nginx-ns
```

## Ingress

```bash
kubectl get ingress -n nginx-ns
kubectl get ingress app-ingress -n nginx-ns -o yaml
kubectl describe ingress app-ingress -n nginx-ns
```

## Ingress Controller

```bash
kubectl get pods -n ingress-nginx -o wide
kubectl get svc -n ingress-nginx
kubectl logs -n ingress-nginx deployment/ingress-nginx-controller
```

## Endpoints

```bash
kubectl get endpoints -n nginx-ns
```

---

# 28. Non-standard host ports, redirects, and my practical experience (While using kubectl port-forward)

While testing the Ingress locally, I learned an important practical issue with using
a non-standard host port.

For example, suppose the Ingress is exposed through:

```text
localhost:81
```

instead of the normal HTTP port:

```text
localhost:80
```

At first, everything can appear to work correctly:

```text
Browser
   │
   │ http://localhost:81
   ▼
Host
   │
   ▼
Kind node
   │
   ▼
Ingress Controller
   │
   ▼
Backend Service
   │
   ▼
Spring Boot
```

The initial request can successfully reach the application.

So it is easy to think:

```text
"Port 81 is working, therefore everything is fine."
```

But the problem can appear when the application returns a **redirect**.

---

## What happened conceptually

Suppose the browser requests:

```text
http://localhost:81/some-path
```

The request reaches the Spring Boot application successfully.

The application then returns a redirect:

```text
HTTP 3xx
Location: <redirected-url>
```

The browser automatically follows that URL.

The important part is that the application may generate the redirected URL based on
the host, scheme, and port that it believes are externally visible.

If the application or reverse-proxy configuration does not correctly know that the
original external port is:

```text
81
```

the generated redirect can contain an unexpected port or scheme.

The flow can then become:

```text
Browser
   │
   │ http://localhost:81
   ▼
Ingress
   │
   ▼
Spring Boot
   │
   │ 3xx redirect
   ▼
Browser follows Location
   │
   ▼
Unexpected / unreachable URL
```

---

## Why this confused me during the lab

The important part of my experience was that the **first request was working**.

That means:

```text
localhost:81
      ↓
Ingress
      ↓
Service
      ↓
Pod
```

was already working.

The problem appeared only after the application redirected the browser.

So it can look like:

```text
"Application was working, then suddenly it stopped."
```

But the actual situation can be:

```text
Application
    │
    │ still healthy
    ▼
returns redirect
    │
    ▼
Browser follows redirect
    │
    ▼
redirect destination is unreachable
```

Therefore, a failed page after a redirect does **not automatically mean that the
Spring Boot application or Kubernetes Pod crashed**.

---

## Example

Imagine the user accesses:

```text
http://localhost:81/login
```

The login request reaches the application.

The application responds with a redirect to another URL.

If the externally visible port is not understood correctly by the application or
proxy, the redirect could point to something such as:

```text
http://localhost:80/...
```

or another unexpected port/scheme.

The browser then tries that new URL.

If nothing is listening there, the browser reports a connection failure.

The important distinction is:

```text
Initial request:
localhost:81 → ✅ reaches application

Redirect request:
redirected URL → ❌ cannot be reached
```

The application itself may still be completely healthy.

---

## Why standard ports are easier for this lab

For a normal HTTP/HTTPS setup, I prefer:

```text
HTTP  → 80
HTTPS → 443
```

because these are the standard ports users and applications commonly expect.

Using:

```text
81
82
8080
```

can still work, but it adds another external-port detail that must be handled
correctly by the proxy and application when redirects or absolute URLs are
involved.

Therefore, for this Kind + Ingress lab, using:

```text
Host :80  → Ingress HTTP
Host :443 → Ingress HTTPS
```

makes the complete request flow easier to understand and troubleshoot.

---

## Practical debugging lesson

When a request works initially but fails after a redirect, do not immediately
assume that the Pod crashed.

Check the actual HTTP response and the redirect target.

For example:

```bash
curl -i http://localhost:81/...
```

Look for:

```text
HTTP/1.1 3xx
Location: ...
```

Then test the redirected URL separately.

Also check the application:

```bash
kubectl get pods -n nginx-ns
```

and:

```bash
kubectl logs -n nginx-ns <backend-pod-name>
```

This helps distinguish between:

```text
Application process failure
```

and:

```text
Correct application response
        ↓
redirect
        ↓
incorrect/unreachable external destination
```

That distinction was one of the important practical lessons from this lab.


---

# 29. Why standard ports are convenient for this lab

Suppose the intended external address is:

```text
https://example.com
```

The normal HTTPS port is:

```text
443
```

If the application is instead accessed through an unusual port, the application, proxy, and browser all need to agree on the externally visible URL.

For a learning lab, the simplest architecture is:

```text
Host :80  → HTTP
Host :443 → HTTPS
```

That makes redirects and absolute URLs much easier to understand.

---

# 30. Complete application architecture

```text
                         Ubuntu Host
                    ┌───────────────────┐
                    │                   │
Browser ───────────►│ :80 / :443       │
                    └─────────┬─────────┘
                              │
                              ▼
                    Kind Docker Node
                              │
                              │ host networking
                              ▼
                 NGINX Ingress Controller
                              │
                    ┌─────────┴─────────┐
                    │                   │
                 /api                  /
                    │                   │
                    ▼                   ▼
             backend-service     nginx-service
                    │                   │
              ┌─────┴─────┐       ┌────┴────┐
              ▼           ▼       ▼         ▼
       backend Pod   backend Pod NGINX Pod NGINX Pod
              │
              ▼
      postgres-service:5432
              │
              ▼
       PostgreSQL Pod
              │
              ▼
          Persistent Storage
```

---

# 31. Main lessons from this lab

## Ingress is not the controller

```text
Ingress
= routing rules

Ingress Controller
= software that processes the rules and handles requests
```

## Kind node is not the host

```text
Ubuntu Host
    ↓
Docker
    ↓
Kind Kubernetes Node
```

Therefore, Kubernetes networking and Ubuntu host networking are separate layers.

## `extraPortMappings` and `hostPort` are different

```text
extraPortMappings
    = Host ↔ Kind node

hostPort
    = Kind node ↔ Pod/container
```

## Scheduling matters with `hostPort`

The `hostPort` belongs to the node where the Pod runs.

Check the real node with:

```bash
kubectl get pods -n ingress-nginx -o wide
```

## Test inside the cluster before testing outside

If this works:

```bash
curl http://backend-service:8080/api
```

and this works:

```bash
curl http://nginx-service:80
```

then the application and Service layers are already working.

## An Ingress path does not create an application resource

```text
Ingress path /nginx
        ≠
NGINX automatically having /nginx
```

The backend must actually know how to handle the forwarded path.

## Standard ports reduce surprises

For a simple browser-facing HTTP/HTTPS lab:

```text
80  → HTTP
443 → HTTPS
```

This makes redirects and externally generated URLs easier to understand.

---

# 32. Practical checklist

When the browser cannot reach the application, check these in order:

```bash
# 1. Application Pods
kubectl get pods -n nginx-ns -o wide

# 2. Services
kubectl get svc -n nginx-ns

# 3. Test Services internally
kubectl run curl-test --rm -it --image=curlimages/curl -n nginx-ns -- sh

# 4. Ingress resource
kubectl get ingress -n nginx-ns

# 5. Ingress Controller location
kubectl get pods -n ingress-nginx -o wide

# 6. Ingress Controller Service
kubectl get svc -n ingress-nginx

# 7. Controller logs
kubectl logs -n ingress-nginx deployment/ingress-nginx-controller

# 8. Kind/Docker networking
kind get clusters
docker ps
```

The key question at each layer is:

```text
Can traffic reach this layer?
```

Do not jump directly to the browser when the earlier layer has not been proven.

---

# 33. Lab conclusion

The final mental model for this setup is:

```text
Host
 ↓
Kind Docker Node
 ↓
Ingress Controller
 ↓
Ingress Rules
 ↓
Service
 ↓
Pod
 ↓
Application
```

For the backend:

```text
/api
 ↓
backend-service:8080
 ↓
Spring Boot Pod:8080
 ↓
PostgreSQL
```

For NGINX:

```text
/
 ↓
nginx-service:80
 ↓
NGINX Pod:80
```

The most important debugging lesson is to identify **which network layer is failing** before changing configuration.