# Kubernetes QoS Classes

Kubernetes assigns every Pod a **Quality of Service (QoS) class** based on the CPU and memory `requests` and `limits` configured for its containers.

There are three QoS classes:

- `BestEffort`
- `Burstable`
- `Guaranteed`

---

## 1. BestEffort

A Pod gets the `BestEffort` class when **no CPU or memory requests or limits are configured**.

Example:

```yaml
containers:
  - name: nginx
    image: nginx:1.27
```

Check the Pod:

```bash
kubectl describe pod <pod-name> -n nginx-ns
```

You will see:

```text
QoS Class: BestEffort
```

### Concept

```text
No CPU requests/limits
No memory requests/limits
        ↓
   BestEffort
```

---

# 2. Burstable

A Pod gets the `Burstable` class when resource configuration exists, but the Pod does not satisfy the requirements for `Guaranteed`.

For example:

```yaml
resources:
  requests:
    cpu: 100m
    memory: 128Mi
  limits:
    cpu: 200m
    memory: 256Mi
```

Here:

```text
CPU request    = 100m
CPU limit      = 200m

Memory request = 128Mi
Memory limit   = 256Mi
```

The requests and limits are different, so Kubernetes assigns:

```text
QoS Class: Burstable
```

Another example that can also result in `Burstable` is configuring only some resources:

```yaml
resources:
  requests:
    cpu: 100m
```

### Concept

```text
Some resource configuration
        ↓
Does not qualify as Guaranteed
        ↓
      Burstable
```

---

# 3. Guaranteed

A Pod gets the `Guaranteed` class when **every container in the Pod** has CPU and memory requests and limits configured, and for each resource:

```text
request == limit
```

Example:

```yaml
resources:
  requests:
    cpu: 200m
    memory: 256Mi
  limits:
    cpu: 200m
    memory: 256Mi
```

Here:

```text
CPU request     = CPU limit
Memory request  = Memory limit
```

So Kubernetes assigns:

```text
QoS Class: Guaranteed
```

### Concept

```text
CPU request = CPU limit
Memory request = Memory limit
for every container
        ↓
    Guaranteed
```

---

# 4. Quick Comparison

| Configuration | QoS Class |
|---|---|
| No CPU/memory requests or limits | `BestEffort` |
| Some resource configuration, but not Guaranteed | `Burstable` |
| CPU + memory requests and limits are equal for every container | `Guaranteed` |

---

# 5. Easy Memory Trick

Remember it like this:

```text
Nothing configured
        ↓
   BestEffort
```

```text
Resources configured
but not fully equal
        ↓
    Burstable
```

```text
CPU request = CPU limit
Memory request = Memory limit
for every container
        ↓
   Guaranteed
```

---

# 6. Your Nginx Example

Your current Deployment uses:

```yaml
resources:
  requests:
    cpu: 100m
    memory: 128Mi
  limits:
    cpu: 200m
    memory: 256Mi
```

Therefore:

```text
QoS Class: Burstable
```

If you remove `resources` completely:

```yaml
containers:
  - name: nginx
    image: nginx:1.27
```

then:

```text
QoS Class: BestEffort
```

If you change it to:

```yaml
resources:
  requests:
    cpu: 200m
    memory: 256Mi
  limits:
    cpu: 200m
    memory: 256Mi
```

then:

```text
QoS Class: Guaranteed
```

---

# 7. Important Detail: QoS Is for the Pod

QoS classification is based on the **entire Pod**, not just one container.

For example, if a Pod has two containers:

```text
Container 1 → correctly configured
Container 2 → missing required resources
```

the Pod may not qualify for `Guaranteed`.

So when checking QoS, think about **all containers inside the Pod**.

---

# 8. Check QoS Class

Use:

```bash
kubectl describe pod <pod-name> -n nginx-ns
```

Look for:

```text
QoS Class: Burstable
```

You can also use:

```bash
kubectl get pod <pod-name> -n nginx-ns -o jsonpath='{.status.qosClass}'
```

Example output:

```text
Burstable
```