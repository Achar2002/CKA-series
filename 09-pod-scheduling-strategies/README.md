# Kubernetes Pod Scheduling Strategies

Kubernetes provides different scheduling strategies to control **where Pods are placed** in a cluster.

---

# 1. Why Control Pod Placement?

By default, the Kubernetes scheduler decides which node should run a Pod.

Sometimes, however, we need more control.

For example, we may want to schedule Pods:

- On nodes with SSD storage
- On GPU-enabled nodes
- In a specific Availability Zone (AZ)
- In a specific region
- On specific types of worker nodes
- Close to related Pods
- Away from other Pods
- To balance workloads
- To improve availability and fault tolerance

---

# 2. Pod Scheduling Strategies

The three important scheduling strategies are:

```text
1. Node Selector
       ↓
   Simple node-based scheduling

2. Node Affinity
       ↓
   Advanced node-based scheduling

3. Pod Affinity / Pod Anti-Affinity
       ↓
   Pod-label-based scheduling
```

### Easy way to remember

```text
Node Selector
    ↓
"Which node?"

Node Affinity
    ↓
"Which nodes, using more advanced rules?"

Pod Affinity
    ↓
"Which Pods should I stay close to?"

Pod Anti-Affinity
    ↓
"Which Pods should I stay away from?"
```

### Analogy

Think of a Kubernetes cluster as a gated community:

```text
Kubernetes Cluster → Gated Community
Nodes              → Individual Flats
Pods                → People living in the flats
```

Scheduling rules decide **which flat a person should live in**.

---

# 3. Node Selector

**Node Selector** is the simplest way to tell Kubernetes to schedule a Pod only on nodes having a specific label.

The basic flow is:

```text
Label the Node
      ↓
nodeSelector in Pod YAML
      ↓
Kubernetes finds matching node
      ↓
Pod gets scheduled there
```

### Important prerequisite

The node must have the required label.

---

# 4. Label a Node

Example:

```bash
kubectl label node <NodeName> env=dev
```

For example:

```bash
kubectl label node ip-192-168-29-16.ec2.internal env=dev
```

Check node labels:

```bash
kubectl get nodes --show-labels
```

---

# 5. Overwrite an Existing Label

If a node already has:

```text
env=dev
```

and we want to change it to:

```text
env=developer
```

use:

```bash
kubectl label node <NodeName> env=developer --overwrite
```

---

# 6. Remove a Node Label

To remove a label:

```bash
kubectl label node <NodeName> env-
```

For example:

```bash
kubectl label node ip-192-168-29-16.ec2.internal env-
```

The `-` after the label key tells Kubernetes to remove that label.

---

# 7. Node Selector Example

Suppose we have:

```text
Node 1
ip-192-168-29-16.ec2.internal
env=dev
```

Our Pod YAML can contain:

```yaml
apiVersion: v1
kind: Pod

metadata:
  name: dev-pod

spec:
  nodeSelector:
    env: dev

  containers:
    - name: nginx
      image: nginx
```

The important part is:

```yaml
spec:
  nodeSelector:
    env: dev
```

Kubernetes will schedule the Pod only on a node having:

```text
env=dev
```

---

# 8. Node Selector Characteristics

### Advantages

- Simple
- Easy to understand
- Easy to configure
- Useful for basic node placement

### Limitation

Node Selector is relatively rigid.

It does not provide advanced operators or preference rules.

For more flexible scheduling, we use **Node Affinity**.

---

# 9. Node Affinity

**Node Affinity** is an advanced and more flexible version of node selection.

It allows us to define:

- Hard requirements
- Soft preferences
- Multiple rules
- Operators

Node Affinity supports operators such as:

```text
In
NotIn
Exists
DoesNotExist
Gt
Lt
```

---

# 10. Hard Rule vs Soft Rule

Node Affinity has two important scheduling approaches.

### Hard Rule

```text
Required
```

The Pod **must** satisfy the rule.

If no matching node exists, the Pod remains unscheduled.

Think:

> "The Pod must run on a node matching this rule."

---

### Soft Rule

```text
Preferred
```

Kubernetes tries to satisfy the rule, but if it cannot, it can schedule the Pod elsewhere.

Think:

> "Try to run the Pod here if possible."

---

# 11. Types of Node Affinity

### Required

```yaml
requiredDuringSchedulingIgnoredDuringExecution
```

This is the **hard requirement**.

The Pod cannot be scheduled unless the rule is satisfied.

### Preferred

```yaml
preferredDuringSchedulingIgnoredDuringExecution
```

This is the **soft preference**.

Kubernetes tries to satisfy the rule but can schedule the Pod elsewhere.

> Note: `preferredDuringSchedulingIgnoredDuringExecution` is the correct spelling. It is `Execution`, not `Executing`.

---

# 12. Node Affinity Example — Required

Example:

```yaml
apiVersion: v1
kind: Pod

metadata:
  name: dev-pod

spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:

          - matchExpressions:
              - key: env
                operator: In
                values:
                  - dev
                  - test

  containers:
    - name: nginx
      image: nginx
```

This means:

```text
Pod can run on:

env=dev
     OR
env=test
```

But it cannot run on a node such as:

```text
env=prod
```

if no other affinity term matches.

---

# 13. Node Affinity — Preferred

Example:

```yaml
apiVersion: v1
kind: Pod

metadata:
  name: preferred-pod

spec:
  affinity:
    nodeAffinity:

      preferredDuringSchedulingIgnoredDuringExecution:

        - weight: 100

          preference:
            matchExpressions:
              - key: env
                operator: In
                values:
                  - dev

  containers:
    - name: nginx
      image: nginx
```

Here Kubernetes **prefers** nodes with:

```text
env=dev
```

But if no such node is available, Kubernetes can choose another suitable node.

---

# 14. Node Affinity Operators

Node Affinity uses `matchExpressions`.

## In

Example:

```yaml
- key: env
  operator: In
  values:
    - test
    - dev
    - prod
```

Meaning:

> The node's `env` label must have one of these values.

Valid:

```text
env=test
env=dev
env=prod
```

---

# 15. NotIn

Example:

```yaml
- key: env
  operator: NotIn
  values:
    - test
```

Meaning:

> The node's `env` value must not be `test`.

So a Pod can be scheduled on nodes such as:

```text
env=dev
env=prod
```

but not:

```text
env=test
```

---

# 16. Exists

Example:

```yaml
- key: gpu
  operator: Exists
```

Meaning:

> The node must have a `gpu` label.

The actual value does not matter.

For example:

```text
gpu=true
```

or:

```text
gpu=nvidia
```

Both satisfy the existence requirement.

---

# 17. DoesNotExist

Example:

```yaml
- key: gpu
  operator: DoesNotExist
```

Meaning:

> The node must not have the `gpu` label.

---

# 18. Gt and Lt

`Gt` means **greater than**.

`Lt` means **less than**.

Example:

```yaml
- key: cpu
  operator: Gt
  values:
    - "4"
```

This can be used when the node label represents a numeric value.

These operators are less commonly used than `In`, `NotIn`, `Exists`, and `DoesNotExist`.

---

# 19. Node Selector vs Node Affinity

| Feature | Node Selector | Node Affinity |
|---|---|---|
| Basic node selection | Yes | Yes |
| Advanced rules | No | Yes |
| Hard requirements | Yes | Yes |
| Soft preferences | No | Yes |
| Operators | Limited | Yes |
| `In` | No | Yes |
| `NotIn` | No | Yes |
| `Exists` | No | Yes |
| `DoesNotExist` | No | Yes |
| Complexity | Simple | More flexible |

Easy way to remember:

```text
Node Selector
    ↓
Simple

Node Affinity
    ↓
Advanced
```

---

# 20. Pod Affinity and Pod Anti-Affinity

Node Selector and Node Affinity make scheduling decisions based on **node labels**.

Pod Affinity and Pod Anti-Affinity make scheduling decisions based on **Pod labels**.

```text
Node Selector / Node Affinity
            ↓
       Node labels


Pod Affinity / Anti-Affinity
            ↓
        Pod labels
```

---

# 21. Pod Affinity

**Pod Affinity** tells Kubernetes to schedule a Pod close to other Pods that match a specified label.

"Close" can mean:

- Same node
- Same zone
- Same region

depending on the `topologyKey`.

---

# 22. Real-World Pod Affinity Example

Imagine we have the Swiggy application:

```text
Swiggy
  |
  ├── Frontend
  ├── Backend
  └── Database
```

Suppose the frontend communicates frequently with the backend.

We may want the frontend Pod to run close to the backend Pod to reduce network latency.

For example:

```text
Node 1
┌─────────────────────┐
│ Swiggy Backend Pod   │
│ Swiggy Frontend Pod  │
└─────────────────────┘
```

We can use **Pod Affinity** to express this placement relationship.

---

# 23. Pod Anti-Affinity

**Pod Anti-Affinity** tells Kubernetes to avoid placing a Pod close to another matching Pod.

This can improve:

- Fault tolerance
- High availability
- Resilience
- Workload distribution

---

# 24. Real-World Pod Anti-Affinity Example

Imagine:

```text
Swiggy Database Pod
Zomato Database Pod
```

If both database Pods are placed on the same node:

```text
Node 1
├── Swiggy DB
└── Zomato DB
```

and Node 1 fails:

```text
Node 1 ❌
   ↓
Both databases unavailable
```

This creates a serious availability problem.

With Pod Anti-Affinity, we can tell Kubernetes to avoid placing those Pods together.

For example:

```text
Node 1
└── Swiggy DB

Node 2
└── Zomato DB
```

Now a failure of Node 1 does not automatically take down both database Pods.

---

# 25. Easy Way to Remember

### Pod Affinity

> **Work near your friends.**

Useful when Pods communicate frequently and low latency is important.

```text
Frontend
   ↕
Backend
```

Keep them close.

### Pod Anti-Affinity

> **Stay away from each other.**

Useful for fault tolerance and workload distribution.

```text
Database A → Node 1
Database B → Node 2
```

Avoid putting both on the same failure domain.

---

# 26. topologyKey

`topologyKey` tells Kubernetes **what topology domain should be considered when determining "close" or "away."**

Common examples include:

```yaml
topologyKey: kubernetes.io/hostname
```

This generally means:

> Same or different node.

For example:

```text
Node 1
Node 2
Node 3
```

Using:

```yaml
topologyKey: kubernetes.io/hostname
```

allows affinity rules to work at the node level.

---

### Zone

```yaml
topologyKey: topology.kubernetes.io/zone
```

This means the scheduling relationship is evaluated at the **Availability Zone** level.

Example:

```text
us-east-1a
us-east-1b
us-east-1c
```

---

### Region

```yaml
topologyKey: topology.kubernetes.io/region
```

This works at the region level.

Example:

```text
us-east-1
us-west-2
```

> Note: The older `failure-domain.beta.kubernetes.io/*` labels have largely been replaced by the stable `topology.kubernetes.io/*` labels.

---

# 27. Scheduling Strategy Comparison

| Strategy | Based On | Main Purpose |
|---|---|---|
| Node Selector | Node labels | Simple node placement |
| Node Affinity | Node labels | Advanced node placement |
| Pod Affinity | Pod labels | Place Pods close together |
| Pod Anti-Affinity | Pod labels | Keep Pods separated |

---

# 28. Overall Scheduling Flow

```text
                    Pod
                     |
          ┌──────────┴──────────┐
          ↓                     ↓
    Node-based              Pod-based
    scheduling              scheduling
          |                     |
     ┌────┴────┐          ┌─────┴─────┐
     ↓         ↓          ↓           ↓
Node Selector  Node      Pod         Pod
               Affinity  Affinity    Anti-Affinity
```

---

# 29. Important Points to Remember

### Node Selector

```text
Simple node-based scheduling
```

Requires:

```text
Node label
     ↓
nodeSelector
```

---

### Node Affinity

```text
Advanced node-based scheduling
```

Supports:

```text
Required → Hard rule
Preferred → Soft rule
```

Common operators:

```text
In
NotIn
Exists
DoesNotExist
Gt
Lt
```

---

### Pod Affinity

```text
Schedule Pods close to matching Pods
```

Useful for:

- Low latency
- Frequently communicating services
- Related workloads

---

### Pod Anti-Affinity

```text
Schedule Pods away from matching Pods
```

Useful for:

- High availability
- Fault tolerance
- Resilience
- Avoiding single-node failures

---

### Final Memory Trick

```text
Node Selector
    ↓
"Put my Pod on this type of node."

Node Affinity
    ↓
"Put my Pod on this type of node,
but I can define advanced rules."

Pod Affinity
    ↓
"Put my Pod close to these Pods."

Pod Anti-Affinity
    ↓
"Keep my Pod away from these Pods."
```
