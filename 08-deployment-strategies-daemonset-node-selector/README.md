# Deployment Rollouts, DaemonSet and Node Selector

This section covers some important Kubernetes topics related to:

- Deployment rollout control
- Rollout pause and resume
- Rollback
- `maxSurge`
- `maxUnavailable`
- DaemonSet
- Node Selector
- Node labels

---

# 1. Deployment Rollout

A Deployment manages application updates using ReplicaSets.

The basic structure is:

```text
Deployment
     |
     ↓
ReplicaSet
     |
     ↓
Pods
```

When the container image is updated, Kubernetes performs a rolling update by default.

---

# 2. Adding Change Cause to a Deployment

We can add an annotation to record why a Deployment was changed.

Example:

```bash
kubectl annotate deployment zomato-deployment \
  kubernetes.io/change-cause="Updated zomato container image to kastrov/swiggy" \
  --overwrite
```

Now the change reason can be seen in the Deployment rollout history.

```bash
kubectl rollout history deployment/zomato-deployment
```

Example:

```text
deployment.apps/zomato-deployment
REVISION  CHANGE-CAUSE
1         <none>
2         Updated zomato container image to kastrov/swiggy
```

This is useful when troubleshooting or deciding which version to roll back to.

---

# 3. Pause a Deployment Rollout

We can temporarily pause a Deployment:

```bash
kubectl rollout pause deployment/zomato-deployment
```

or:

```bash
kubectl rollout pause deploy/zomato-deployment
```

When a Deployment is paused, Kubernetes does not continue applying further changes to the Deployment.

For example, if we make multiple changes while the Deployment is paused, we can group those changes and apply them after resuming.

### Important

A paused Deployment does not mean that existing Pods stop running.

The existing application continues running.

---

# 4. Resume a Deployment

To continue the rollout:

```bash
kubectl rollout resume deployment/zomato-deployment
```

or:

```bash
kubectl rollout resume deploy/zomato-deployment
```

After resuming, Kubernetes continues reconciling the Deployment.

---

# 5. Rollback a Deployment

If the new application version has a problem, we can roll back to the previous version.

```bash
kubectl rollout undo deployment/zomato-deployment
```

Check the rollout:

```bash
kubectl rollout status deployment/zomato-deployment
```

Check rollout history:

```bash
kubectl rollout history deployment/zomato-deployment
```

The Deployment manages ReplicaSets, so Kubernetes can use the previous ReplicaSet during rollback.

---

# 6. maxSurge

`maxSurge` controls how many **additional Pods** can be created above the desired number of Pods during a rolling update.

Example:

```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxSurge: 1
```

Suppose we have:

```text
replicas: 3
maxSurge: 1
```

Normally:

```text
3 Pods
```

During the update, Kubernetes can temporarily create:

```text
3 old Pods + 1 new Pod
                ↑
             maxSurge
```

So the temporary maximum becomes:

```text
4 Pods
```

`maxSurge` can be specified as:

### Absolute number

```yaml
maxSurge: 1
```

or:

```yaml
maxSurge: 2
```

### Percentage

```yaml
maxSurge: 50%
```

For example, with 4 desired Pods:

```text
4 × 50% = 2
```

So Kubernetes can temporarily have up to:

```text
6 Pods
```

during the rollout.

---

# 7. maxUnavailable

`maxUnavailable` controls how many Pods can be unavailable during a rolling update.

Example:

```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxUnavailable: 1
```

This means Kubernetes can have at most 1 Pod unavailable during the update.

It can also be specified as:

### Absolute number

```yaml
maxUnavailable: 1
```

### Percentage

```yaml
maxUnavailable: 50%
```

---

# 8. maxSurge + maxUnavailable Example

Suppose we have:

```yaml
replicas: 3
```

and:

```yaml
maxSurge: 1
maxUnavailable: 1
```

Initially:

```text
Old Pods

Pod 1
Pod 2
Pod 3
```

### Step 1

A new Pod can be created because:

```text
maxSurge = 1
```

Now:

```text
Old Pod 1
Old Pod 2
Old Pod 3
New Pod 1
```

Total:

```text
4 Pods
```

### Step 2

Once the new Pod is ready, Kubernetes can terminate an old Pod.

```text
Old Pod 1
Old Pod 2
       ❌ Old Pod 3
New Pod 1
```

### Step 3

Another new Pod is created:

```text
Old Pod 1
Old Pod 2
New Pod 1
New Pod 2
```

The process continues until all old Pods have been replaced.

Finally:

```text
New Pod 1
New Pod 2
New Pod 3
```

The temporary extra Pod does **not** remain after the rollout.

---

# 9. Why maxSurge and maxUnavailable Are Important

These settings control the balance between:

```text
Application availability
        ↕
Deployment speed
        ↕
Available resources
```

For example:

```yaml
maxSurge: 1
maxUnavailable: 0
```

This prioritizes availability because Kubernetes should not make an existing Pod unavailable before a replacement is ready.

A common production-style configuration might be:

```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxSurge: 1
    maxUnavailable: 0
```

The exact values depend on the application's availability requirements and available cluster resources.

---

# 10. DaemonSet

A **DaemonSet** is a Kubernetes controller that ensures a Pod runs on each eligible node.

For example:

```text
Node 1 → DaemonSet Pod
Node 2 → DaemonSet Pod
Node 3 → DaemonSet Pod
```

If a new eligible node is added:

```text
Node 4 → DaemonSet Pod
```

Kubernetes automatically schedules a DaemonSet Pod on that node.

If a node is removed, the Pod associated with that node is also removed.

---

# 11. DaemonSet vs Deployment

DaemonSet and Deployment are **different controllers**.

### Deployment

Used when we want a specific number of application replicas.

Example:

```text
replicas: 3

Node 1 → Pod
Node 2 → Pod
Node 3 → Pod
```

The scheduler can place those Pods on suitable nodes.

### DaemonSet

Used when we want a Pod on every eligible node.

```text
Node 1 → Pod
Node 2 → Pod
Node 3 → Pod
```

The number of Pods is determined by the eligible nodes rather than a `replicas` field.

### Easy way to remember

```text
Deployment
    ↓
"I want 3 Pods"

DaemonSet
    ↓
"I want 1 Pod on every eligible node"
```

---

# 12. Real-World Uses of DaemonSet

DaemonSets are commonly used for node-level services.

Examples:

### Log collection

A log agent can run on every node and collect container logs.

```text
Node 1 → Log Agent
Node 2 → Log Agent
Node 3 → Log Agent
```

### Monitoring

A monitoring agent can run on every node and collect node-level metrics.

### Networking

Some Kubernetes networking components use DaemonSets to run components on nodes.

### Security agents

Security or runtime monitoring agents can be deployed to every node.

---

# 13. Example DaemonSet

```yaml
apiVersion: apps/v1
kind: DaemonSet

metadata:
  name: log-agent

spec:
  selector:
    matchLabels:
      app: log-agent

  template:
    metadata:
      labels:
        app: log-agent

    spec:
      containers:
        - name: log-agent
          image: fluent/fluent-bit
```

The DaemonSet ensures that a Pod is scheduled on each eligible node.

---

# 14. Node Selector

Sometimes we don't want a Pod to run on every node.

We may want to run it on a **specific node**.

For this, we can use a **Node Selector**.

The basic process is:

```text
Label the Node
      ↓
Create nodeSelector
      ↓
Pod gets scheduled on matching node
```

---

# 15. Labeling Nodes

Suppose our nodes are:

```text
ip-192-168-11-218.ec2.internal → worker1

ip-192-168-44-170.ec2.internal → worker2
```

We can assign labels to them.

Example:

```bash
kubectl label node ip-192-168-11-218.ec2.internal role=worker1
```

And:

```bash
kubectl label node ip-192-168-44-170.ec2.internal role=worker2
```

Check the labels:

```bash
kubectl get nodes --show-labels
```

Or:

```bash
kubectl get nodes -l role=worker1
```

---

# 16. Using nodeSelector

Now we can tell Kubernetes to schedule a Pod on the node with:

```text
role=worker1
```

Example:

```yaml
apiVersion: apps/v1
kind: Deployment

metadata:
  name: demo-deployment

spec:
  replicas: 3

  selector:
    matchLabels:
      app: demo

  template:
    metadata:
      labels:
        app: demo

    spec:
      nodeSelector:
        role: worker1

      containers:
        - name: demo-container
          image: nginx
```

The important part is:

```yaml
nodeSelector:
  role: worker1
```

Kubernetes will schedule the Pods only on nodes having:

```text
role=worker1
```

---

# 17. Node Selector Flow

```text
Node 1
role=worker1
        \
         \
          → Pod scheduled here
         /
        /
Node 2
role=worker2
```

The Pod has:

```yaml
nodeSelector:
  role: worker1
```

Therefore:

```text
Pod → Node 1
```

and not:

```text
Pod → Node 2
```

---

# 18. Useful Commands

### View nodes

```bash
kubectl get nodes
```

### View node labels

```bash
kubectl get nodes --show-labels
```

### Add a label

```bash
kubectl label node <node-name> role=worker1
```

### View nodes matching a label

```bash
kubectl get nodes -l role=worker1
```

### View DaemonSets

```bash
kubectl get daemonsets
```

or:

```bash
kubectl get ds
```

### Describe a DaemonSet

```bash
kubectl describe daemonset <daemonset-name>
```

### View Deployment rollout status

```bash
kubectl rollout status deployment/<deployment-name>
```

### View rollout history

```bash
kubectl rollout history deployment/<deployment-name>
```

### Pause rollout

```bash
kubectl rollout pause deployment/<deployment-name>
```

### Resume rollout

```bash
kubectl rollout resume deployment/<deployment-name>
```

### Roll back

```bash
kubectl rollout undo deployment/<deployment-name>
```

---

# 19. Key Points to Remember

### Deployment

```text
Deployment
    ↓
ReplicaSet
    ↓
Pods
```

Used for:

- Application deployments
- Scaling
- Rolling updates
- Rollbacks

### maxSurge

Controls the number of **extra Pods** allowed during a rolling update.

```text
maxSurge = Extra Pods
```

### maxUnavailable

Controls the number of Pods that can be unavailable during the update.

```text
maxUnavailable = Allowed unavailable Pods
```

### DaemonSet

```text
1 Pod → Every eligible node
```

Used commonly for:

- Logging agents
- Monitoring agents
- Networking components
- Security agents

### Node Selector

Used to schedule Pods on nodes having a specific label.

```text
Node Label
    ↓
nodeSelector
    ↓
Pod scheduled on matching node
```

The overall concept to remember is:

```text
Deployment
    ↓
Controls application versions
    ↓
ReplicaSets
    ↓
Pods


DaemonSet
    ↓
Runs a Pod on every eligible node


nodeSelector
    ↓
Controls which node a Pod can run on
```
