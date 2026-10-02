# ReplicationController, ReplicaSet and Deployment

Kubernetes provides different controllers to manage Pods and maintain the desired state of an application.

The main controllers discussed in this section are:

- ReplicationController (RC)
- ReplicaSet (RS)
- Deployment
- DaemonSet
- StatefulSet

---

## 1. ReplicationController (RC)

A **ReplicationController** ensures that a specified number of Pod replicas are running at all times.

If a Pod crashes or gets deleted, the ReplicationController creates a replacement Pod.

### Example

```yaml
apiVersion: v1
kind: ReplicationController

metadata:
  name: zomato-rc

spec:
  replicas: 3

  selector:
    app: zomato

  template:
    metadata:
      labels:
        app: zomato

    spec:
      containers:
        - name: zomato
          image: zomato:1.0
```

Here:

```text
ReplicationController
        |
        | manages
        ↓
     3 Pods
```

### Scaling an RC

We can increase or decrease the number of Pods:

```bash
kubectl scale rc zomato-rc --replicas=5
```

Check the Pods:

```bash
kubectl get pods
```

### Limitations of ReplicationController

ReplicationController is a **legacy controller** and has been superseded by ReplicaSet and Deployment for modern applications.

Main limitations:

- Older/legacy controller
- Supports only equality-based selectors
- Does not perform rolling updates
- Does not provide built-in rollback functionality
- Not normally used for new applications

---

# 2. ReplicaSet (RS)

A **ReplicaSet** is the newer controller used to maintain a desired number of identical Pods.

It provides improved selector capabilities compared with ReplicationController.

### API Version

ReplicaSet uses:

```yaml
apiVersion: apps/v1
```

### Example

```yaml
apiVersion: apps/v1
kind: ReplicaSet

metadata:
  name: zomato-rs

spec:
  replicas: 3

  selector:
    matchLabels:
      app: zomato

  template:
    metadata:
      labels:
        app: zomato

    spec:
      containers:
        - name: zomato
          image: zomato:1.0
```

The relationship is:

```text
ReplicaSet
     |
     | manages
     ↓
   Pods
```

---

## 3. What is a Selector?

A **selector** is used by Kubernetes controllers to identify Pods based on their labels.

For example, a Pod can have:

```yaml
labels:
  app: zomato
```

And a ReplicaSet can select it using:

```yaml
selector:
  matchLabels:
    app: zomato
```

The label on the Pod must match the selector used by the controller.

---

## Types of Selectors

### Equality-Based Selector

ReplicationController supports equality-based selection.

Example:

```yaml
selector:
  app: zomato
```

This means:

```text
app = zomato
```

---

### Set-Based Selector

ReplicaSet supports richer selector expressions.

Example:

```yaml
selector:
  matchLabels:
    app: swiggy
    env: dev
    version: v1.1
```

A ReplicaSet can also use `matchExpressions` for set-based selection:

```yaml
selector:
  matchExpressions:
    - key: env
      operator: In
      values:
        - dev
        - test
```

This provides more flexible Pod selection.

---

# 4. ReplicationController vs ReplicaSet

| Feature | ReplicationController | ReplicaSet |
|---|---|---|
| Purpose | Maintain Pod replicas | Maintain Pod replicas |
| Self-healing | Yes | Yes |
| Scaling | Yes | Yes |
| Selector | Equality-based | Equality + set-based |
| API Version | `v1` | `apps/v1` |
| Rolling Updates | No | No |
| Rollback | No | No |
| Modern usage | Legacy | Used by Deployments |

### Important

ReplicaSet itself **does not perform rolling updates**.

A ReplicaSet manages a set of Pods. For rolling updates and rollbacks, we normally use a **Deployment**.

---

# 5. Deployment

A **Deployment** is the standard Kubernetes controller used to manage stateless applications.

A Deployment helps us:

- Maintain the desired number of Pods
- Scale Pods
- Perform rolling updates
- Roll back to a previous version
- Manage ReplicaSets
- Maintain application availability during updates

The hierarchy is:

```text
Deployment
     |
     | manages
     ↓
ReplicaSet
     |
     | manages
     ↓
   Pods
```

When we create a Deployment, Kubernetes automatically creates a ReplicaSet.

---

## Example Deployment

```yaml
apiVersion: apps/v1
kind: Deployment

metadata:
  name: zomato-deployment

spec:
  replicas: 3

  selector:
    matchLabels:
      app: zomato

  template:
    metadata:
      labels:
        app: zomato

    spec:
      containers:
        - name: zomato
          image: zomato:1.0
          ports:
            - containerPort: 8080
```

When this Deployment is created:

```text
Deployment
     |
     ↓
ReplicaSet
     |
     ↓
3 Pods
```

---

# 6. Rolling Update

A **rolling update** gradually replaces Pods running the old application version with Pods running the new version.

For example:

```text
Version 1
Zomato:1.0
     |
     ↓
Deployment
     |
     ↓
ReplicaSet v1
     |
     ↓
Pods
```

Now we update the image:

```text
Zomato:1.0
       ↓
Zomato:2.0
```

The Deployment creates a new ReplicaSet:

```text
Deployment
     |
     ├── ReplicaSet v1
     │      └── Old Pods
     │
     └── ReplicaSet v2
            └── New Pods
```

The new Pods are gradually created while the old Pods are gradually removed.

This allows the application to remain available during the update when the Deployment is configured appropriately.

### Check rollout status

```bash
kubectl rollout status deployment/zomato-deployment
```

---

# 7. Recreate Strategy

The **Recreate** strategy works differently.

It first removes the existing Pods and then creates new Pods with the updated application version.

```text
Old Pods
   ↓
Delete all old Pods
   ↓
Create new Pods
   ↓
New version
```

Because there is a period when the old Pods are gone and the new Pods are not ready yet, this strategy can cause **downtime**.

Example:

```yaml
strategy:
  type: Recreate
```

---

# 8. Deployment Strategies

Kubernetes Deployments directly support these strategy types:

### RollingUpdate

Default strategy.

```yaml
strategy:
  type: RollingUpdate
```

Old Pods are gradually replaced with new Pods.

### Recreate

```yaml
strategy:
  type: Recreate
```

Old Pods are removed before new Pods are created.

---

## Blue-Green Deployment

Blue-Green is a common deployment/release pattern.

```text
Blue  → Current version
Green → New version
```

Both versions can be maintained separately, and traffic can be switched between them.

Blue-Green deployment generally requires additional Service/Ingress configuration or deployment tooling; it is not simply a third `Deployment.spec.strategy.type`.

---

## Canary Deployment

In a Canary deployment, the new version is released to a small portion of traffic/users first.

Example:

```text
90% → Version 1
10% → Version 2
```

If the new version behaves correctly, traffic can gradually be increased.

Canary deployments usually require additional configuration or tools for traffic management.

---

# 9. Rollback

One of the major advantages of a Deployment is that it can keep previous ReplicaSets and allow us to roll back to an earlier revision.

### Check rollout history

```bash
kubectl rollout history deployment/zomato-deployment
```

### Roll back

```bash
kubectl rollout undo deployment/zomato-deployment
```

### Check rollout status

```bash
kubectl rollout status deployment/zomato-deployment
```

---

# 10. Scaling a Deployment

We can increase or decrease the number of Pods.

Example:

```bash
kubectl scale deployment zomato-deployment --replicas=5
```

Check:

```bash
kubectl get pods
```

---

# 11. Checking the Image Used by Pods

We can check the image running in a Pod:

```bash
kubectl describe pod <pod-name> | grep -i Image
```

We can also select Pods using labels:

```bash
kubectl get pods -l app=zomato
```

For example:

```bash
kubectl describe pod -l app=zomato | grep -i Image
```

---

# 12. Editing a ReplicaSet

We can edit a ReplicaSet using:

```bash
kubectl edit rs/zomato-rs
```

However, when a ReplicaSet is created and managed by a **Deployment**, we generally should not directly modify the ReplicaSet.

Instead, modify the **Deployment**, because:

```text
Deployment
     |
     ↓
ReplicaSet
     |
     ↓
Pods
```

The Deployment is the owner and manages the ReplicaSet.

For example, to update the application image:

```bash
kubectl set image deployment/zomato-deployment zomato=zomato:2.0
```

This causes the Deployment to create/manage the appropriate ReplicaSet for the new version.

---

# 13. RC vs RS vs Deployment

| Feature | RC | RS | Deployment |
|---|---|---|---|
| Maintain replicas | Yes | Yes | Yes |
| Auto-healing | Yes | Yes | Yes |
| Scaling | Yes | Yes | Yes |
| Equality selectors | Yes | Yes | Yes |
| Set-based selectors | No | Yes | Yes |
| Rolling Updates | No | No | Yes |
| Rollback | No | No | Yes |
| Manages ReplicaSet | No | No | Yes |
| Modern usage | Legacy | Usually managed by Deployment | Standard |

---

# 14. Key Difference

The easiest way to remember the relationship:

```text
ReplicationController
        ↓
     Manages
        ↓
      Pods
```

```text
ReplicaSet
    ↓
 Manages
    ↓
  Pods
```

```text
Deployment
      ↓
 Manages
      ↓
 ReplicaSet
      ↓
    Pods
```

Therefore:

> **Deployment → ReplicaSet → Pods**

This is the most commonly used structure for deploying stateless applications in Kubernetes.

---

# 15. Important Points to Remember

- ReplicationController is a legacy controller.
- ReplicaSet provides improved Pod selector capabilities.
- Both RC and RS maintain the desired number of Pods.
- ReplicaSet does **not** perform rolling updates by itself.
- Deployment manages ReplicaSets.
- Deployment performs rolling updates by default.
- Deployment supports rollback.
- Deployment can scale applications.
- Deployment creates and manages ReplicaSets automatically.
- Blue-Green and Canary are common deployment patterns that require additional configuration/tools.
- When a ReplicaSet is managed by a Deployment, modify the Deployment rather than directly editing the ReplicaSet.
