# Kubernetes Taints and Tolerations — Practical Guide

This README explains **Kubernetes taints and tolerations** from beginner level to practical usage.

The goal is to understand:

- What a taint is
- Why Kubernetes uses taints
- How to add, view, and remove taints
- What `key`, `operator`, `value`, and `effect` mean
- What `NoSchedule`, `PreferNoSchedule`, and `NoExecute` do
- What a toleration is
- How `Equal` and `Exists` work
- How `tolerationSeconds` works with `NoExecute`
- How to test everything using a simple Deployment
- Common mistakes and useful commands

---

## 1. The simplest mental model

Think about a **Kubernetes Node as a machine** and a **Pod as an application that wants to run on that machine**.

A taint is like putting a sign on a node:

> **"Do not place normal pods here."**

A toleration is like giving a pod permission to ignore that particular sign:

> **"This pod is allowed to stay on this node."**

So:

```text
Node
  |
  |-- Taint: dedicated=gpu:NoSchedule
  |
  |       means
  |       "Pods that do not tolerate this taint should not be scheduled here"
  |
  +----> Pod A
          No matching toleration
          -> cannot be scheduled here

  +----> Pod B
          Matching toleration
          -> can be scheduled here
```

### Important

A taint is placed on a **node**.

A toleration is placed on a **pod**.

```text
NODE  -> TAINT
POD   -> TOLERATION
```

A toleration does **not** force a pod onto a node. It only says that the pod is allowed to run there.

This distinction is extremely important.

---

# 2. Why do we need taints and tolerations?

Suppose a Kubernetes cluster has these nodes:

```text
node-1   normal application node
node-2   GPU node
node-3   database/dedicated node
```

You do not want every normal application pod to randomly use the GPU node or database-dedicated node.

You can taint the special node.

For example:

```bash
kubectl taint nodes node-2 workload=gpu:NoSchedule
```

Now Kubernetes sees:

```text
node-2
  workload=gpu:NoSchedule
```

A pod without a matching toleration will not be scheduled there.

A GPU workload can have a matching toleration:

```yaml
tolerations:
  - key: "workload"
    operator: "Equal"
    value: "gpu"
    effect: "NoSchedule"
```

Now that pod can be scheduled on the tainted node.

---

# 3. Taint syntax

The basic command is:

```bash
kubectl taint nodes <node-name> <key>=<value>:<effect>
```

Example:

```bash
kubectl taint nodes worker-1 workload=gpu:NoSchedule
```

Break it into pieces:

```text
kubectl taint nodes worker-1 workload=gpu:NoSchedule
                 |        |       |     |
                 |        |       |     +---- effect
                 |        |       +---------- value
                 |        +------------------ key
                 +--------------------------- node name
```

---

# 4. Understanding `key`

Example:

```text
workload=gpu:NoSchedule
```

Here:

```text
workload
```

is the **key**.

The key is simply a name that describes why/how you are tainting the node.

Common examples:

```text
dedicated
workload
environment
team
hardware
gpu
storage
```

There is no magical meaning to a key by itself.

For example, these are different taints:

```text
workload=gpu:NoSchedule
workload=database:NoSchedule
```

Same key, different values.

---

# 5. Understanding `value`

Example:

```text
workload=gpu:NoSchedule
```

Here:

```text
gpu
```

is the **value**.

The value gives additional information about the key.

For example:

```text
workload=gpu
workload=database
workload=batch
```

The Kubernetes scheduler can match a pod's toleration against the node's taint using the key, operator, value, and effect.

---

# 6. Understanding `effect`

This is the part you should understand very well.

Kubernetes supports three taint effects:

```text
NoSchedule
PreferNoSchedule
NoExecute
```

Each one behaves differently.

---

## 6.1 `NoSchedule`

Example:

```bash
kubectl taint nodes worker-1 workload=gpu:NoSchedule
```

Meaning:

> Do not schedule new pods onto this node unless they tolerate the taint.

Example situation:

```text
worker-1
  workload=gpu:NoSchedule
```

Pod A:

```text
No matching toleration
```

Result:

```text
Pod A -> not scheduled onto worker-1
```

Pod B:

```yaml
tolerations:
  - key: "workload"
    operator: "Equal"
    value: "gpu"
    effect: "NoSchedule"
```

Result:

```text
Pod B -> allowed onto worker-1
```

### Important behavior

`NoSchedule` mainly affects **new scheduling decisions**.

If a pod is already running on a node and you add a `NoSchedule` taint, Kubernetes does not use that taint by itself to immediately evict the already-running pod.

---

# 7. `PreferNoSchedule`

Example:

```bash
kubectl taint nodes worker-1 workload=gpu:PreferNoSchedule
```

This is a **soft preference**.

Meaning:

> Try not to place pods on this node if possible, but this is not an absolute prohibition.

Think of it as:

```text
NoSchedule
  -> "DO NOT put normal pods here"

PreferNoSchedule
  -> "Please avoid putting normal pods here"
```

The scheduler may still place a pod there when other scheduling constraints or cluster conditions make that necessary.

So:

```text
PreferNoSchedule = preference, not a hard block
```

---

# 8. `NoExecute`

Example:

```bash
kubectl taint nodes worker-1 workload=gpu:NoExecute
```

This has a stronger effect.

It means:

1. New pods that do not tolerate the taint should not be scheduled onto the node.
2. Pods already running there that do not tolerate the taint can be evicted.

Think:

```text
NoSchedule
  -> don't place new matching pods here

NoExecute
  -> don't place new pods here
  -> remove existing pods that don't tolerate the taint
```

This is why `NoExecute` is commonly encountered in situations where Kubernetes needs workloads removed from a node.

---

# 9. Quick comparison of the three effects

| Effect | New pod can be scheduled without toleration? | Existing pod without toleration can remain? |
|---|---|---|
| `NoSchedule` | No | Yes |
| `PreferNoSchedule` | Usually avoid, but may happen | Yes |
| `NoExecute` | No | No, it can be evicted |

The word **"can"** matters here because actual scheduling also depends on other constraints such as selectors, affinity, resources, topology, and node availability.

---

# 10. Add a taint to a node

First see your nodes:

```bash
kubectl get nodes
```

Example:

```text
NAME                  STATUS   ROLES           AGE   VERSION
my-cluster-control    Ready    control-plane   ...   v1.31.2
my-cluster-worker     Ready    <none>          ...   v1.31.2
my-cluster-worker2    Ready    <none>          ...   v1.31.2
```

Choose a worker node.

Example:

```bash
kubectl taint nodes my-cluster-worker workload=gpu:NoSchedule
```

You should see something similar to:

```text
node/my-cluster-worker tainted
```

---

# 11. Check the taint

Use:

```bash
kubectl describe node my-cluster-worker
```

Look for:

```text
Taints: workload=gpu:NoSchedule
```

You can also inspect the node as YAML:

```bash
kubectl get node my-cluster-worker -o yaml
```

Look under:

```yaml
spec:
  taints:
    - key: workload
      value: gpu
      effect: NoSchedule
```

---

# 12. Remove a taint

Use the taint followed by a `-`.

For example, if you added:

```bash
kubectl taint nodes my-cluster-worker workload=gpu:NoSchedule
```

Remove it with:

```bash
kubectl taint nodes my-cluster-worker workload=gpu:NoSchedule-
```

The important part is:

```text
NoSchedule-
```

The trailing `-` means **remove this taint**.

You can verify:

```bash
kubectl describe node my-cluster-worker
```

---

# 13. Multiple taints on one node

A node can have multiple taints.

Example:

```bash
kubectl taint nodes worker-1 workload=gpu:NoSchedule
kubectl taint nodes worker-1 environment=prod:NoExecute
```

Now the node has:

```text
workload=gpu:NoSchedule
environment=prod:NoExecute
```

A pod generally needs to tolerate all taints that would otherwise block its scheduling or continued execution on that node.

Think of a node with two locks:

```text
Lock 1 -> workload=gpu
Lock 2 -> environment=prod
```

The pod needs appropriate tolerations for both.

---

# 14. What is a toleration?

A toleration is placed inside the pod specification.

Example:

```yaml
spec:
  tolerations:
    - key: "workload"
      operator: "Equal"
      value: "gpu"
      effect: "NoSchedule"
```

This tells Kubernetes:

> This pod tolerates the node taint `workload=gpu:NoSchedule`.

Remember:

```text
Taint      -> Node
Toleration  -> Pod
```

---

# 15. Toleration fields

A toleration commonly contains:

```yaml
tolerations:
  - key: "workload"
    operator: "Equal"
    value: "gpu"
    effect: "NoSchedule"
```

The important fields are:

```text
key
operator
value
effect
tolerationSeconds
```

`operator` controls how Kubernetes matches the toleration against the taint.

---

# 16. `operator: Equal`

Example:

```yaml
- key: "workload"
  operator: "Equal"
  value: "gpu"
  effect: "NoSchedule"
```

This is saying:

```text
key must match
AND
value must match
AND
effect must match
```

Node taint:

```text
workload=gpu:NoSchedule
```

Pod toleration:

```text
workload + Equal + gpu + NoSchedule
```

Result:

```text
MATCH
```

But if the node has:

```text
workload=database:NoSchedule
```

then the value does not match:

```text
gpu != database
```

Therefore this toleration does not tolerate that taint.

### Another example

Taint:

```text
dedicated=backend:NoSchedule
```

Toleration:

```yaml
- key: "dedicated"
  operator: "Equal"
  value: "backend"
  effect: "NoSchedule"
```

Matches.

---

# 17. `operator: Exists`

`Exists` is different.

Example:

```yaml
- key: "workload"
  operator: "Exists"
  effect: "NoSchedule"
```

Meaning:

> If a taint with this key and this effect exists, the pod tolerates it regardless of its value.

For example, this toleration can match:

```text
workload=gpu:NoSchedule
```

and also:

```text
workload=database:NoSchedule
```

and also:

```text
workload=batch:NoSchedule
```

because the key is:

```text
workload
```

and the operator says:

```text
Exists
```

### Important rule

With `Exists`, do **not** specify a value.

Correct:

```yaml
- key: "workload"
  operator: "Exists"
  effect: "NoSchedule"
```

Not this:

```yaml
- key: "workload"
  operator: "Exists"
  value: "gpu"
  effect: "NoSchedule"
```

---

# 18. `Equal` vs `Exists`

This is one of the most important concepts.

### `Equal`

```yaml
- key: "workload"
  operator: "Equal"
  value: "gpu"
  effect: "NoSchedule"
```

Means:

```text
workload=gpu
```

Only the specific value matches.

### `Exists`

```yaml
- key: "workload"
  operator: "Exists"
  effect: "NoSchedule"
```

Means:

```text
workload=<any value>
```

The value does not matter.

### Easy memory trick

```text
Equal  -> "I tolerate THIS value"
Exists -> "I tolerate THIS key, whatever the value is"
```

---

# 19. What about the `effect` inside a toleration?

Example:

```yaml
- key: "workload"
  operator: "Equal"
  value: "gpu"
  effect: "NoSchedule"
```

The toleration's effect matches the taint's effect.

Node:

```text
workload=gpu:NoSchedule
```

Pod:

```text
workload=gpu:NoSchedule
```

Match.

If the node has:

```text
workload=gpu:NoExecute
```

then the above toleration is not a matching `NoExecute` toleration.

You can also omit `effect` in a toleration to make it match taints with that key/operator/value across all effects.

Example:

```yaml
- key: "workload"
  operator: "Equal"
  value: "gpu"
```

This can match a taint with the same key/value regardless of whether the effect is `NoSchedule`, `PreferNoSchedule`, or `NoExecute`.

For learning and production readability, explicitly specifying the intended effect is often clearer.

---

# 20. Special case: empty key with `Exists`

There is a special wildcard-style case.

Example:

```yaml
- operator: "Exists"
```

With an empty key and `Exists`, the toleration can match taints regardless of their key.

This is very broad.

Use it carefully because it effectively says:

> I tolerate taints of the relevant effect even when I do not care what the key is.

For learning purposes, it is better to first understand explicit keys such as:

```yaml
- key: "workload"
  operator: "Exists"
  effect: "NoSchedule"
```

before using the broad form.

---

# 21. A complete working example

We will create:

```text
Node
  |
  +-- taint: workload=gpu:NoSchedule

Pod
  |
  +-- toleration: workload=gpu:NoSchedule
```

## Step 1 — List nodes

```bash
kubectl get nodes
```

Choose a worker node, for example:

```text
kind-worker
```

## Step 2 — Add the taint

```bash
kubectl taint nodes kind-worker workload=gpu:NoSchedule
```

## Step 3 — Verify it

```bash
kubectl describe node kind-worker
```

You should find:

```text
Taints: workload=gpu:NoSchedule
```

---

# 22. Create a pod WITHOUT a toleration

Create `nginx-no-toleration.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-no-toleration
spec:
  containers:
    - name: nginx
      image: nginx:latest
```

Apply it:

```bash
kubectl apply -f nginx-no-toleration.yaml
```

Check:

```bash
kubectl get pod nginx-no-toleration -o wide
```

Depending on the other nodes and scheduling constraints in your cluster, the pod may simply schedule onto another suitable node. If the tainted node is the only suitable node, the pod will remain unscheduled.

To understand why a pending pod cannot be scheduled, inspect it:

```bash
kubectl describe pod nginx-no-toleration
```

Look at the `Events` section.

You may see a message indicating that the node's taint is not tolerated.

---

# 23. Create a pod WITH a toleration

Create `nginx-with-toleration.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-with-toleration
spec:
  tolerations:
    - key: "workload"
      operator: "Equal"
      value: "gpu"
      effect: "NoSchedule"
  containers:
    - name: nginx
      image: nginx:latest
```

Apply:

```bash
kubectl apply -f nginx-with-toleration.yaml
```

Check:

```bash
kubectl get pod nginx-with-toleration -o wide
```

The pod is now **allowed** to run on the tainted node.

### Very important

Toleration does NOT mean:

> "Put this pod on that node."

It only means:

> "This pod is allowed to run on a node with this taint."

If you want to specifically target a particular node, you also need a scheduling mechanism such as `nodeSelector` or node affinity.

---

# 24. Toleration + nodeSelector

Suppose you want a pod to run on your GPU node specifically.

First label the node:

```bash
kubectl label node kind-worker workload=gpu
```

Then taint it:

```bash
kubectl taint node kind-worker workload=gpu:NoSchedule
```

Now the pod can use both:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: gpu-nginx
spec:
  nodeSelector:
    workload: gpu

  tolerations:
    - key: "workload"
      operator: "Equal"
      value: "gpu"
      effect: "NoSchedule"

  containers:
    - name: nginx
      image: nginx:latest
```

The two settings have different jobs:

```text
nodeSelector
  -> tells scheduler WHERE the pod should go

 toleration
  -> tells scheduler WHICH taint the pod is allowed to tolerate
```

This is a very common pattern for dedicated nodes.

---

# 25. `NoExecute` + `tolerationSeconds`

This is an important advanced part.

Suppose a node has:

```text
dedicated=backend:NoExecute
```

A pod can tolerate it permanently:

```yaml
tolerations:
  - key: "dedicated"
    operator: "Equal"
    value: "backend"
    effect: "NoExecute"
```

Or for a limited amount of time:

```yaml
tolerations:
  - key: "dedicated"
    operator: "Equal"
    value: "backend"
    effect: "NoExecute"
    tolerationSeconds: 60
```

This means:

> The pod can tolerate the `NoExecute` taint for 60 seconds after the taint is applied (subject to Kubernetes pod/node behavior and other lifecycle constraints).

Conceptually:

```text
Node gets NoExecute taint
        |
        v
Pod continues to tolerate it
        |
        v
60 seconds pass
        |
        v
Pod is eligible for eviction
```

Without `tolerationSeconds`, the matching `NoExecute` toleration has no expiration.

---

# 26. Example of `NoExecute`

Taint the node:

```bash
kubectl taint nodes kind-worker maintenance=true:NoExecute
```

Pod with 30-second tolerance:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: maintenance-demo
spec:
  tolerations:
    - key: "maintenance"
      operator: "Equal"
      value: "true"
      effect: "NoExecute"
      tolerationSeconds: 30
  containers:
    - name: nginx
      image: nginx:latest
```

The pod can temporarily remain on the tainted node, but once its tolerance expires, it can be evicted.

---

# 27. Commands you should remember

## List nodes

```bash
kubectl get nodes
```

## Describe a node

```bash
kubectl describe node <node-name>
```

## Add a `NoSchedule` taint

```bash
kubectl taint nodes <node-name> key=value:NoSchedule
```

## Add a `PreferNoSchedule` taint

```bash
kubectl taint nodes <node-name> key=value:PreferNoSchedule
```

## Add a `NoExecute` taint

```bash
kubectl taint nodes <node-name> key=value:NoExecute
```

## Remove a taint

```bash
kubectl taint nodes <node-name> key=value:NoSchedule-
```

For the other effects:

```bash
kubectl taint nodes <node-name> key=value:PreferNoSchedule-
kubectl taint nodes <node-name> key=value:NoExecute-
```

## Show node YAML

```bash
kubectl get node <node-name> -o yaml
```

## Check pod placement

```bash
kubectl get pods -o wide
```

## See why a pod is pending

```bash
kubectl describe pod <pod-name>
```

---

# 28. Tainting with only a key and no value

A taint can also have no value.

Example:

```bash
kubectl taint nodes worker-1 dedicated:NoSchedule
```

This represents a taint similar to:

```text
dedicated:NoSchedule
```

A matching toleration can use `Exists`:

```yaml
tolerations:
  - key: "dedicated"
    operator: "Exists"
    effect: "NoSchedule"
```

This is one reason it is useful to understand `Exists` instead of assuming every taint must look like:

```text
key=value:effect
```

---

# 29. Example: dedicated database node

Imagine you have a database node:

```text
db-node-1
```

You do not want normal application workloads there.

Taint the node:

```bash
kubectl taint node db-node-1 dedicated=database:NoSchedule
```

A database pod can tolerate it:

```yaml
tolerations:
  - key: "dedicated"
    operator: "Equal"
    value: "database"
    effect: "NoSchedule"
```

This creates a clean separation:

```text
Normal applications
       X
       |
       | blocked by taint
       v
+----------------------+
| db-node-1            |
| dedicated=database   |
| NoSchedule            |
+----------------------+
       ^
       |
       | allowed with toleration
       |
 Database workload
```

In real clusters, this is often combined with labels and node affinity/selectors so that the intended workloads are both **allowed** and **targeted** to the dedicated nodes.

---

# 30. What happens when a pod has multiple tolerations?

Example:

```yaml
tolerations:
  - key: "workload"
    operator: "Equal"
    value: "gpu"
    effect: "NoSchedule"

  - key: "environment"
    operator: "Equal"
    value: "prod"
    effect: "NoSchedule"
```

Suppose the node has:

```text
workload=gpu:NoSchedule
environment=prod:NoSchedule
```

The pod has matching tolerations for both.

Therefore it can tolerate both taints.

Conceptually:

```text
Node taint #1 -> tolerated
Node taint #2 -> tolerated

Result -> pod is allowed from a taint/toleration perspective
```

Again, other scheduling constraints can still prevent placement.

---

# 31. Taints do not reserve a node by themselves

This is a common beginner mistake.

Suppose:

```bash
kubectl taint node worker-1 workload=gpu:NoSchedule
```

and a pod has:

```yaml
- key: "workload"
  operator: "Equal"
  value: "gpu"
  effect: "NoSchedule"
```

It does **not** guarantee:

```text
Pod -> worker-1
```

It only means:

```text
Pod -> is allowed on worker-1
```

To actually target that node, combine the toleration with:

- `nodeSelector`, or
- node affinity

---

# 32. Taint vs label

This distinction is extremely useful in Kubernetes.

### Label

A node label helps **identify/select** nodes.

Example:

```bash
kubectl label node worker-1 workload=gpu
```

Then:

```yaml
nodeSelector:
  workload: gpu
```

means roughly:

> I want a node with this label.

### Taint

A taint is used to **repel** workloads unless they have a matching toleration.

Example:

```bash
kubectl taint node worker-1 workload=gpu:NoSchedule
```

means roughly:

> Keep ordinary pods away from this node.

### Together

A common design is:

```text
LABEL
workload=gpu
   |
   +--> attracts/selects the intended workload

TAINT
workload=gpu:NoSchedule
   |
   +--> repels workloads that should not be there
```

This gives you both sides:

```text
Node affinity / nodeSelector -> "come here"
Taint + toleration           -> "you are allowed here"
```

---

# 33. A simple real-world scenario

Imagine three nodes:

```text
worker-1 -> normal
worker-2 -> GPU
worker-3 -> database
```

Configure them like this:

```bash
kubectl label node worker-2 workload=gpu
kubectl taint node worker-2 workload=gpu:NoSchedule

kubectl label node worker-3 workload=database
kubectl taint node worker-3 workload=database:NoSchedule
```

Now:

```text
worker-1
  normal workloads

worker-2
  label: workload=gpu
  taint: workload=gpu:NoSchedule

worker-3
  label: workload=database
  taint: workload=database:NoSchedule
```

GPU workload:

```yaml
nodeSelector:
  workload: gpu

tolerations:
  - key: "workload"
    operator: "Equal"
    value: "gpu"
    effect: "NoSchedule"
```

Database workload:

```yaml
nodeSelector:
  workload: database

tolerations:
  - key: "workload"
    operator: "Equal"
    value: "database"
    effect: "NoSchedule"
```

Now the cluster expresses two requirements:

```text
1. Find the right kind of node.
2. Make sure the pod is permitted to run there.
```

---

# 34. Common mistakes

## Mistake 1 — Thinking toleration forces placement

Wrong mental model:

```text
toleration = send pod to node
```

Correct:

```text
toleration = pod is allowed to run on matching tainted node
```

Use a selector or affinity to target a node.

---

## Mistake 2 — Mixing up node and pod

Remember:

```text
taint      -> node
```

```text
toleration -> pod
```

---

## Mistake 3 — Using `Exists` with a value

Correct:

```yaml
operator: "Exists"
```

without `value`.

Use `Equal` when you need an exact key/value match.

---

## Mistake 4 — Confusing `NoSchedule` and `NoExecute`

```text
NoSchedule
  -> blocks new scheduling

NoExecute
  -> blocks new scheduling AND can evict existing pods
```

---

## Mistake 5 — Forgetting the effect

This taint:

```text
workload=gpu:NoSchedule
```

and this toleration:

```yaml
- key: "workload"
  operator: "Equal"
  value: "gpu"
  effect: "NoExecute"
```

are not equivalent for effect matching.

Use the effect that corresponds to the taint you intend to tolerate, or omit `effect` when you deliberately want the toleration to match all effects for that key/operator/value combination.

---

# 35. Useful inspection commands

Show all nodes:

```bash
kubectl get nodes
```

Show labels:

```bash
kubectl get nodes --show-labels
```

Show taints in a compact custom column:

```bash
kubectl get nodes -o custom-columns=NAME:.metadata.name,TAINTS:.spec.taints
```

Describe a node:

```bash
kubectl describe node <node-name>
```

Show pod node placement:

```bash
kubectl get pods -o wide
```

Inspect a pod's scheduling events:

```bash
kubectl describe pod <pod-name>
```

---

# 36. A small practice lab

Use this lab to learn the concept instead of memorizing it.

## Step 1 — Get nodes

```bash
kubectl get nodes
```

## Step 2 — Select a worker node

```bash
kubectl get nodes -o wide
```

Suppose your node is:

```text
kind-worker
```

## Step 3 — Add a taint

```bash
kubectl taint node kind-worker dedicated=lab:NoSchedule
```

## Step 4 — Verify

```bash
kubectl describe node kind-worker
```

Find:

```text
Taints: dedicated=lab:NoSchedule
```

## Step 5 — Create a pod without tolerance

```bash
kubectl run test-no-toleration --image=nginx
```

Check:

```bash
kubectl get pod test-no-toleration -o wide
```

If another node is available, Kubernetes may schedule the pod there.

## Step 6 — Inspect scheduling events

```bash
kubectl describe pod test-no-toleration
```

## Step 7 — Create a pod with tolerance

Create `lab-toleration.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-with-toleration
spec:
  tolerations:
    - key: "dedicated"
      operator: "Equal"
      value: "lab"
      effect: "NoSchedule"

  containers:
    - name: nginx
      image: nginx:latest
```

Apply:

```bash
kubectl apply -f lab-toleration.yaml
```

Check:

```bash
kubectl get pod test-with-toleration -o wide
```

## Step 8 — Remove the taint

```bash
kubectl taint node kind-worker dedicated=lab:NoSchedule-
```

## Step 9 — Clean up pods

```bash
kubectl delete pod test-no-toleration test-with-toleration
```

---

# 37. One-page mental model

Remember this diagram:

```text
                         KUBERNETES
                              |
                 +------------+------------+
                 |                         |
               NODE                       POD
                 |                         |
               TAINT                  TOLERATION
                 |                         |
       +---------+---------+       +-------+-------+
       |         |         |       |       |       |
      key      value     effect   key   operator  value
                                      |
                                   effect
                                      |
                              tolerationSeconds
```

### Taint

```text
key=value:effect
```

Example:

```text
workload=gpu:NoSchedule
```

### Toleration

```yaml
- key: "workload"
  operator: "Equal"
  value: "gpu"
  effect: "NoSchedule"
```

### Operators

```text
Equal
  -> key + value match

Exists
  -> key exists; value is ignored
```

### Effects

```text
NoSchedule
  -> don't schedule new pods unless tolerated

PreferNoSchedule
  -> avoid scheduling there when possible

NoExecute
  -> don't schedule there and can evict existing non-tolerating pods
```

### Very important

```text
Taint      = repels
Toleration = permits
Selector/Affinity = helps target
```

---

# 38. Interview-ready explanation

### What is a taint?

A taint is a property applied to a Kubernetes node that tells the scheduler to repel pods unless those pods have a matching toleration.

### What is a toleration?

A toleration is a pod specification that allows the pod to be scheduled onto a node with a matching taint. It does not force the pod onto that node.

### What are the three taint effects?

```text
NoSchedule
PreferNoSchedule
NoExecute
```

`NoSchedule` blocks new scheduling without a matching toleration.

`PreferNoSchedule` is a soft avoidance preference.

`NoExecute` can also cause existing non-tolerating pods to be evicted.

### What is the difference between `Equal` and `Exists`?

`Equal` matches a specific key/value pair.

`Exists` matches the key without requiring a value.

### Does toleration force pod placement?

No.

A toleration only makes the pod eligible with respect to the matching taint. Node selectors or node affinity can be used when you need to target particular nodes.

---

# 39. Final cheat sheet

```text
TAINT COMMAND

kubectl taint node NODE KEY=VALUE:EFFECT

Example:
kubectl taint node kind-worker workload=gpu:NoSchedule
```

```text
REMOVE TAINT

kubectl taint node NODE KEY=VALUE:EFFECT-

Example:
kubectl taint node kind-worker workload=gpu:NoSchedule-
```

```text
TAINT

workload=gpu:NoSchedule
   |
   +-- key    = workload
   +-- value  = gpu
   +-- effect = NoSchedule
```

```text
TOLERATION

- key: workload
  operator: Equal
  value: gpu
  effect: NoSchedule
```

```text
OPERATORS

Equal  -> exact value
Exists -> any value for that key
```

```text
EFFECTS

NoSchedule         -> block new scheduling
PreferNoSchedule   -> avoid if possible
NoExecute          -> block new scheduling + evict non-tolerating pods
```

```text
CORE RULE

Node has taint
       |
       v
Pod needs matching toleration
       |
       v
Pod is allowed from taint/toleration perspective
       |
       v
Selector / affinity may be used to target the node
```

---

## Recommended practice order

Learn and practice these in exactly this order:

```text
1. NoSchedule
2. Equal
3. Exists
4. Remove taint
5. Multiple taints/tolerations
6. NoExecute
7. tolerationSeconds
8. Taint + label + nodeSelector
9. Taint + node affinity
```

Once these are clear, taints and tolerations become much easier to reason about in real Kubernetes clusters.