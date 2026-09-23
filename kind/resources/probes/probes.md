# Kubernetes Liveness, Readiness and Startup Probes

Kubernetes probes are health checks performed by the **kubelet** to monitor containers.

There are three important probes:

* **Liveness Probe** → checks whether the container is still alive.
* **Readiness Probe** → checks whether the container is ready to receive traffic.
* **Startup Probe** → checks whether the application has finished starting.

---

# 1. Liveness Probe

A liveness probe answers:

> **"Is this container still alive and working?"**

Example:

```yaml
livenessProbe:
  httpGet:
    path: /
    port: 80
```

Kubernetes does **not** run this only once.

It runs the probe **periodically**.

By default:

```text
periodSeconds = 10
```

So the flow is approximately:

```text
Container starts
      ↓
Liveness check
      ↓
Wait
      ↓
Liveness check
      ↓
Wait
      ↓
Liveness check
      ↓
...
```

If the liveness probe repeatedly fails according to the configured failure threshold:

```text
Liveness probe fails
        ↓
Container considered unhealthy
        ↓
Kubernetes restarts the container
```

### Important

A liveness probe is **similar to a repeated/cron-style check**, but it is **not a Kubernetes CronJob**.

The **kubelet** performs the checks continuously.

---

# 2. Readiness Probe

A readiness probe answers:

> **"Can this container receive traffic right now?"**

Example:

```yaml
readinessProbe:
  httpGet:
    path: /
    port: 80
```

Readiness is also **checked periodically**.

It is **not only checked when the container starts**.

The flow is:

```text
Container starts
      ↓
Readiness check
      ↓
Wait
      ↓
Readiness check
      ↓
Wait
      ↓
Readiness check
      ↓
...
```

If the readiness probe fails:

```text
Readiness probe fails
        ↓
Pod becomes NOT READY
        ↓
Service stops sending traffic to that Pod
```

The container is **not automatically restarted just because readiness failed**.

---

# 3. Startup Probe

Kubernetes also has a third probe called the **startup probe**.

A startup probe answers:

> **"Has my application finished starting?"**

Example:

```yaml
startupProbe:
  httpGet:
    path: /
    port: 80
```

A startup probe is useful for applications that take a long time to start.

For example:

```text
Spring Boot application
        ↓
Container starts
        ↓
Application takes 60 seconds to initialize
        ↓
Startup Probe keeps checking
        ↓
Application becomes available
        ↓
Startup Probe succeeds
```

## How Startup Probe Runs

A startup probe is also a **repeated check**, but it is specifically used during the application's startup phase.

For example:

```yaml
startupProbe:
  httpGet:
    path: /
    port: 80
  periodSeconds: 10
  failureThreshold: 30
```

Conceptually:

```text
Container starts
      ↓
Startup check
      ↓
Fails
      ↓
Wait 10 seconds
      ↓
Startup check
      ↓
Fails
      ↓
Wait 10 seconds
      ↓
Startup check
      ↓
Succeeds
      ↓
Startup completed
```

Once the startup probe succeeds, the startup probe **stops running for that container instance**.

---

## Startup Probe and Liveness/Readiness

When a `startupProbe` is configured, Kubernetes uses it to protect the application during startup.

The important flow is:

```text
Container starts
       ↓
 Startup Probe
       ↓
 ┌─────┴─────┐
 │           │
FAIL       SUCCESS
 │           │
 │           ↓
 │    Liveness + Readiness
 │           │
 │           ↓
 │      Normal operation
 │
 └── Keep checking
```

While the startup probe has **not succeeded**, Kubernetes does not run the liveness and readiness probes for that container.

This prevents a slow-starting application from being restarted by its liveness probe before it has finished starting.

---

# 4. Why Do We Need a Startup Probe?

Imagine a Spring Boot application takes 60 seconds to start.

Without a startup probe:

```text
Container starts
      ↓
Liveness starts checking
      ↓
Application is still starting
      ↓
Liveness fails
      ↓
Container restarts
      ↓
Application starts again
      ↓
Liveness fails again
      ↓
Container restarts
```

This can create a restart loop.

With a startup probe:

```text
Container starts
      ↓
Startup Probe checks
      ↓
Application is still starting
      ↓
Keep checking
      ↓
Application finishes starting
      ↓
Startup Probe succeeds
      ↓
Liveness + Readiness begin
```

So the startup probe gives a slow-starting application time to initialize.

---

# 5. Main Difference

The easiest way to remember them:

```text
STARTUP
"Have you finished starting?"
       ↓
NO
       ↓
Keep checking
```

```text
LIVENESS
"Are you alive?"
       ↓
NO
       ↓
Restart container
```

```text
READINESS
"Are you ready for traffic?"
       ↓
NO
       ↓
Stop sending traffic
```

| Probe     | Question                               | When it runs                      | Failure result             |
| --------- | -------------------------------------- | --------------------------------- | -------------------------- |
| Startup   | Has the application finished starting? | Repeatedly until startup succeeds | Container can be restarted |
| Liveness  | Is the container alive?                | Periodically                      | Container can be restarted |
| Readiness | Can the container receive traffic?     | Periodically                      | Pod becomes Not Ready      |

---

# 6. Do Probes Run Only at Startup?

No.

The three probes have different behavior.

### Liveness

Runs periodically:

```text
Liveness → check → wait → check → wait → check → ...
```

### Readiness

Runs periodically:

```text
Readiness → check → wait → check → wait → check → ...
```

### Startup

Runs repeatedly **during startup**:

```text
Startup → check → wait → check → wait → check
                                      ↓
                                  succeeds
                                      ↓
                              Startup stops
```

After the startup probe succeeds, liveness and readiness can run normally.

---

# 7. `initialDelaySeconds`

Unless `initialDelaySeconds` is configured, a probe can begin checking without an explicit startup delay.

Example:

```yaml
livenessProbe:
  httpGet:
    path: /
    port: 80
  initialDelaySeconds: 10
```

This means Kubernetes waits approximately 10 seconds before starting that probe.

The same concept can be used for readiness:

```yaml
readinessProbe:
  httpGet:
    path: /
    port: 80
  initialDelaySeconds: 10
```

### Startup Probe vs `initialDelaySeconds`

These solve slightly different problems.

`initialDelaySeconds`:

```text
"Wait this fixed amount of time before checking."
```

`startupProbe`:

```text
"Keep checking until the application actually starts."
```

For applications with unpredictable or long startup times, a startup probe can be more appropriate than simply guessing a fixed delay.

---

# 8. Example Deployment

For a simple nginx application:

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

          startupProbe:
            httpGet:
              path: /
              port: 80
            periodSeconds: 10
            failureThreshold: 30

          livenessProbe:
            httpGet:
              path: /
              port: 80

          readinessProbe:
            httpGet:
              path: /
              port: 80

          resources:
            requests:
              cpu: 100m
              memory: 128Mi

            limits:
              cpu: 200m
              memory: 256Mi
```

For this simple nginx container, a startup probe is usually **not necessary** because nginx starts very quickly.

It is more useful for applications such as:

* Spring Boot applications
* Large Java applications
* Applications that perform migrations during startup
* Applications that load large models or data
* Applications that require significant initialization

---

# 9. How to Check Probe Configuration

Use:

```bash
kubectl describe pod <pod-name> -n nginx-ns
```

You may see:

```text
Startup:   http-get http://:80/
Liveness:  http-get http://:80/
Readiness: http-get http://:80/
```

This confirms that the probes are configured.

You can also check:

```bash
kubectl get pods -n nginx-ns
```

Example:

```text
NAME                         READY   STATUS    RESTARTS
nginx-depl-6c67cdb966-q9dln  1/1     Running   0
nginx-depl-6c67cdb966-vdf8w  1/1     Running   0
```

### `READY 1/1`

This indicates that the container is currently **Ready**.

### `RESTARTS 0`

This means the container has not been restarted.

---

# 10. Testing Readiness

To understand readiness properly, imagine the application is temporarily unable to serve requests.

The readiness probe starts failing:

```text
Readiness failure
      ↓
Pod becomes:
READY 0/1
      ↓
Service does not send new traffic
      ↓
Container continues running
```

So:

```text
READY 0/1
```

does **not necessarily mean the container has stopped**.

The container may still show:

```text
STATUS = Running
```

---

# 11. Testing Liveness

For learning, you can intentionally stop nginx inside a Pod.

First find a Pod:

```bash
kubectl get pods -n nginx-ns
```

Then:

```bash
kubectl exec -it <pod-name> -n nginx-ns -- nginx -s stop
```

Watch the Pod:

```bash
kubectl get pods -n nginx-ns -w
```

Because nginx is no longer responding, the liveness probe will eventually fail.

Then Kubernetes can restart the container.

You may see:

```text
NAME                         READY   STATUS    RESTARTS
nginx-depl-6c67cdb966-q9dln  1/1     Running   1
```

The important part is:

```text
RESTARTS = 1
```

That demonstrates the liveness probe caused Kubernetes to detect the unhealthy container and restart it.

---

# 12. Testing Startup Probe

Startup probes are easiest to understand with a slow-starting application.

For example:

```yaml
startupProbe:
  httpGet:
    path: /
    port: 80
  periodSeconds: 5
  failureThreshold: 12
```

This gives the application approximately:

```text
5 seconds × 12 failures = 60 seconds
```

to successfully start before Kubernetes considers the startup probe failed.

Conceptually:

```text
Container starts
      ↓
Startup Probe
      ↓
FAIL → wait 5s
      ↓
Startup Probe
      ↓
FAIL → wait 5s
      ↓
Startup Probe
      ↓
...
      ↓
SUCCESS
      ↓
Startup complete
      ↓
Liveness + Readiness take over
```

If it reaches the configured failure threshold without succeeding:

```text
Startup Probe repeatedly fails
        ↓
Container considered unhealthy
        ↓
Kubernetes restarts container
```

---

# 13. Easy Memory Trick

Remember these three questions:

```text
STARTUP
"Have you finished starting?"
        ↓
NO
        ↓
Keep checking
```

```text
LIVENESS
"Are you alive?"
        ↓
NO
        ↓
Restart
```

```text
READINESS
"Can you handle traffic?"
        ↓
NO
        ↓
Stop sending traffic
```

The overall flow is:

```text
             Container Starts
                    │
                    ▼
              Startup Probe
                    │
             ┌──────┴──────┐
             │             │
           Fails         Succeeds
             │             │
             │             ▼
             │      Liveness + Readiness
             │             │
             │       ┌─────┴─────┐
             │       │           │
             │    Liveness    Readiness
             │       │           │
             │       ▼           ▼
             │    Restart     Traffic on/off
             │
             └── Keep checking
```

> **Startup protects slow-starting applications.**
>
> **Liveness detects containers that are no longer healthy.**
>
> **Readiness controls whether a Pod should receive traffic.**

---

# 14. One-Line Summary

```text
Startup   → "Wait until I finish starting."
Liveness  → "Restart me if I become unhealthy."
Readiness → "Don't send traffic to me if I'm not ready."
```

> **Startup is mainly about initialization, liveness is about being alive, and readiness is about receiving traffic.**
