# 06 - Kubernetes Namespaces and Controllers

## 1. Kubernetes Namespaces

A **Namespace** is a way to logically divide and organize resources within a Kubernetes cluster.

Namespaces are useful when multiple teams, applications, or environments share the same Kubernetes cluster.

For example:

```text
Kubernetes Cluster
│
├── dev namespace
│   ├── Pods
│   ├── Services
│   └── Deployments
│
└── test namespace
    ├── Pods
    ├── Services
    └── Deployments
```

Namespaces provide **logical isolation and organization of resources**.

---

# 2. Why Use Namespaces?

Namespaces are useful for:

- Separating different teams
- Separating environments such as Dev and Test
- Organizing resources
- Applying resource quotas and policies
- Applying access control using RBAC

Example:

```text
Dev Team
   |
   v
dev namespace
   |
   ├── Pods
   ├── Services
   └── Deployments


Test Team
   |
   v
test namespace
   |
   ├── Pods
   ├── Services
   └── Deployments
```

---

# 3. Default Kubernetes Namespaces

When a Kubernetes cluster is created, several namespaces are normally created automatically.

Check them using:

```bash
kubectl get namespaces
```

or:

```bash
kubectl get ns
```

Common default namespaces include:

| Namespace | Purpose |
|---|---|
| `default` | Default namespace for resources when no namespace is specified |
| `kube-system` | Kubernetes system components |
| `kube-public` | Resources that should be publicly readable within the cluster |
| `kube-node-lease` | Contains Lease objects used for node heartbeats |

---

# 4. Default Namespace

If we create a namespaced resource without specifying a namespace, Kubernetes normally creates it in the `default` namespace.

Example:

```bash
kubectl run nginx --image=nginx
```

Check:

```bash
kubectl get pods
```

This looks for Pods in the current/default namespace.

---

# 5. Working with a Specific Namespace

To list Pods in a particular namespace:

```bash
kubectl get pods -n dev
```

To create a resource in a specific namespace:

```bash
kubectl create namespace dev
```

Then:

```bash
kubectl run nginx --image=nginx -n dev
```

Check:

```bash
kubectl get pods -n dev
```

---

# 6. Namespace and RBAC

Namespaces can be combined with **RBAC (Role-Based Access Control)** to control which users can access which resources.

Example:

```text
Dev Team
├── dev1
├── dev2
└── dev3
      |
      v
   dev namespace


Test Team
├── test1
└── test2
      |
      v
   test namespace
```

The organization can configure RBAC so that:

- Dev users have access to resources in the `dev` namespace.
- Test users have access to resources in the `test` namespace.
- Users do not automatically get access to every namespace.

> Namespace isolation by itself does not provide security or access control. RBAC is used to control user permissions.

---

# 7. Kubernetes Controllers

So far, we have worked with:

```text
Pods → Services → Namespaces
```

However, creating a Pod directly does not provide the desired level of application self-healing.

Consider this situation:

```text
Pod
 |
 v
Application
```

If someone deletes the Pod:

```text
Pod
 |
 X
Deleted
```

The Pod is gone.

A standalone Pod does not automatically create a replacement Pod simply because it was deleted.

This is where **Kubernetes Controllers** become important.

---

# 8. What is a Kubernetes Controller?

A Kubernetes **Controller** is a control loop that watches the current state of resources and works to make the current state match the desired state.

The basic idea is:

```text
Desired State
     |
     v
Kubernetes Controller
     |
     v
Current State
```

The controller continuously observes the cluster and performs reconciliation when the current state differs from the desired state.

---

# 9. Desired State vs Current State

Suppose we declare:

```yaml
replicas: 3
```

This means we want **3 Pod replicas**.

Initially:

```text
Desired State = 3 Pods
Current State = 3 Pods
```

Everything is correct.

Now suppose one Pod is deleted:

```text
Desired State = 3 Pods
Current State = 2 Pods
```

The controller detects the difference.

It reconciles the state:

```text
Desired: 3
Current: 2
        |
        v
Controller
        |
        v
Creates another Pod
        |
        v
Current: 3
```

This process is called **reconciliation**.

---

# 10. What is Reconciliation?

**Reconciliation** is the process of continuously comparing the desired state with the current state and taking action to make them match.

Example:

```text
Desired State
3 Pods
   |
   v
Current State
2 Pods
   |
   v
Controller detects difference
   |
   v
Creates 1 Pod
   |
   v
Current State
3 Pods
```

The controller continues monitoring the resources.

---

# 11. Why Controllers Are Important

Controllers provide important capabilities such as:

- Self-healing
- Maintaining the desired number of Pods
- Scaling applications
- Managing application rollouts
- Replacing failed or deleted Pods
- Maintaining the desired state

This helps Kubernetes provide **high availability and self-healing behavior** for applications managed by controllers.

---

# 12. `replicas`

The `replicas` field specifies how many copies of a Pod should be maintained.

Example:

```yaml
spec:
  replicas: 3
```

This means:

```text
Desired number of Pods = 3
```

If one Pod disappears:

```text
3 desired
2 current
```

The controller creates another Pod to return to:

```text
3 desired
3 current
```

---

# 13. Types of Kubernetes Controllers

Some important Kubernetes controllers include:

1. ReplicationController
2. ReplicaSet
3. Deployment
4. DaemonSet
5. StatefulSet

Each controller has a different purpose.

---

# 14. ReplicationController

A **ReplicationController (RC)** ensures that a specified number of Pod replicas are running.

For example:

```text
replicas: 3
```

The ReplicationController attempts to maintain:

```text
3 running Pods
```

If a Pod is deleted or fails:

```text
3 Pods
 ↓
1 Pod deleted
 ↓
2 Pods remain
 ↓
ReplicationController detects difference
 ↓
Creates replacement Pod
 ↓
3 Pods
```

---

# 15. Scaling with ReplicationController

ReplicationController can also be used to scale the number of Pod replicas.

For example:

```text
Current:
3 Pods
```

Scale up:

```text
3 → 5 Pods
```

The controller creates additional Pods.

Scale down:

```text
5 → 2 Pods
```

The controller removes Pods until the desired replica count is reached.

---

# 16. ReplicationController - Legacy Controller

ReplicationController is an older Kubernetes controller.

It has largely been replaced by **ReplicaSet**, which provides similar functionality with more flexible selectors.

In modern Kubernetes workloads, Deployments and ReplicaSets are much more commonly used than ReplicationControllers.

For CKA preparation, it is still important to understand what a ReplicationController does and why it exists.

---

# 17. Controller Architecture

A simplified Kubernetes architecture can be represented as:

```text
                 Desired State
                      |
                      v
               Kubernetes API
                      |
                      v
                 Controller
                      |
                Reconciliation
                      |
                      v
               Current State
                      |
                      v
                    Pods
```

The controller continuously works toward the desired state.

---

# 18. Namespace + Controller + Pod

These concepts can work together:

```text
Kubernetes Cluster
│
└── dev namespace
      │
      └── Deployment
            │
            └── ReplicaSet
                  │
             ┌────┼────┐
             │    │    │
             v    v    v
           Pod  Pod  Pod
```

The controller hierarchy will be explored further when learning **ReplicaSets and Deployments**.

---

# 19. Useful Commands

### List namespaces

```bash
kubectl get namespaces
```

### Create a namespace

```bash
kubectl create namespace dev
```

### List Pods in a namespace

```bash
kubectl get pods -n dev
```

### List all Pods across namespaces

```bash
kubectl get pods -A
```

### Describe a namespace

```bash
kubectl describe namespace dev
```

### Delete a namespace

```bash
kubectl delete namespace dev
```

> Deleting a namespace also deletes the namespaced resources inside it, so use this command carefully.

---

# 20. Key Takeaways

- Namespaces logically organize resources inside a Kubernetes cluster.
- Common default namespaces include `default`, `kube-system`, `kube-public`, and `kube-node-lease`.
- RBAC can be used with namespaces to control user access.
- A standalone Pod does not automatically recreate itself when deleted.
- Controllers provide reconciliation and self-healing behavior for managed workloads.
- Controllers continuously compare desired state with current state.
- The `replicas` field defines the desired number of Pod replicas.
- Reconciliation brings the current state back toward the desired state.
- ReplicationController maintains a desired number of Pod replicas.
- ReplicationController is a legacy controller.
- ReplicaSet and Deployment are more commonly used in modern Kubernetes.
- Controllers are an important part of Kubernetes' declarative and self-healing model.

---

## What I Practiced

- Understanding Kubernetes namespaces
- Listing and creating namespaces
- Working with resources inside specific namespaces
- Understanding the purpose of RBAC with namespaces
- Understanding Kubernetes Controllers
- Understanding desired state and current state
- Understanding reconciliation
- Understanding the `replicas` concept
- Understanding ReplicationController
- Understanding Pod self-healing through controllers
- Understanding scaling with ReplicationController
