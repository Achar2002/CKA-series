# Kubernetes Resource Quotas, Limit Ranges and Autoscaling

This section covers how Kubernetes controls resource consumption and automatically adjusts workloads based on resource usage.

Topics covered:

- Resource Requests
- Resource Limits
- ResourceQuota
- LimitRange
- CPU and Memory units
- Object count quotas
- Horizontal Pod Autoscaler (HPA)
- Vertical Pod Autoscaler (VPA)
- Cluster Autoscaler (CA)
- Metrics Server

---

# 1. Resource Management in Kubernetes

A Kubernetes cluster contains resources such as:

- CPU
- Memory
- Pods
- Services
- PersistentVolumeClaims
- Secrets
- ConfigMaps
- Other Kubernetes objects

Without resource controls, one team or application could consume a large amount of the available resources.

Kubernetes provides mechanisms to control this.

```text
Cluster
   |
   ├── Namespace A
   │      ├── Pod
   │      ├── Pod
   │      └── Pod
   │
   └── Namespace B
          ├── Pod
          └── Pod
```

We can control resource usage at different levels.

---

# 2. Resource Requests

A **resource request** specifies the minimum amount of CPU or memory that a container requires.

Example:

```yaml
resources:
  requests:
    cpu: 200m
    memory: 200Mi
```

This means:

```text
CPU:
200m = 0.2 CPU

Memory:
200Mi
```

The Kubernetes scheduler uses resource requests when deciding which node has enough available capacity to schedule the Pod.

### Easy way to remember

```text
Request
   ↓
"What does this container need?"
```

---

# 3. Resource Limits

A **resource limit** specifies the maximum amount of CPU or memory that a container is allowed to use.

Example:

```yaml
resources:
  limits:
    cpu: 500m
    memory: 500Mi
```

This means:

```text
CPU → maximum 0.5 CPU
Memory → maximum 500Mi
```

### Easy way to remember

```text
Limit
   ↓
"What is the maximum this container can use?"
```

---

# 4. Requests and Limits Together

Example:

```yaml
resources:
  requests:
    cpu: 200m
    memory: 200Mi

  limits:
    cpu: 500m
    memory: 500Mi
```

Here:

```text
CPU:
Request = 200m
Limit   = 500m

Memory:
Request = 200Mi
Limit   = 500Mi
```

For a container, the request cannot be greater than its corresponding limit when both are specified.

A request can be equal to the limit.

Example:

```yaml
requests:
  cpu: 500m

limits:
  cpu: 500m
```

---

# 5. What Happens If Requests and Limits Are Not Specified?

If no resource requests or limits are defined, Kubernetes does not automatically impose per-container CPU and memory limits merely because the Pod exists.

However, a namespace can have a **LimitRange** that automatically supplies defaults.

Also, Kubernetes can apply other mechanisms such as namespace-level quotas.

Therefore, always check the namespace configuration before assuming that a Pod has unlimited resources.

---

# 6. CPU in Kubernetes

CPU is measured in CPU cores.

Kubernetes also supports **millicores**.

```text
1 CPU = 1000m
```

Examples:

```text
100m  = 0.1 CPU
200m  = 0.2 CPU
500m  = 0.5 CPU
1000m = 1 CPU
2000m = 2 CPUs
```

Example:

```yaml
resources:
  requests:
    cpu: 200m

  limits:
    cpu: 500m
```

This means:

```text
Minimum requested CPU → 0.2 CPU
Maximum CPU → 0.5 CPU
```

---

# 7. Why Use Millicores?

Kubernetes commonly runs many containers on the same node.

Instead of assigning an entire CPU core to every small application, we can allocate fractions of CPU.

For example:

```text
Node
│
├── Application A → 200m
├── Application B → 500m
├── Application C → 300m
└── Application D → 100m
```

This allows more efficient resource utilization.

---

# 8. Memory in Kubernetes

Memory is specified using units such as:

```text
Ki
Mi
Gi
```

Common conversions:

```text
1 Mi = 1024 Ki
1 Gi = 1024 Mi
```

Example:

```yaml
resources:
  requests:
    memory: 200Mi

  limits:
    memory: 500Mi
```

This means:

```text
Minimum requested memory → 200Mi
Maximum memory → 500Mi
```

---

# 9. M vs Mi

There is an important difference between:

```text
M  → Megabytes
Mi → Mebibytes
```

Similarly:

```text
G  → Gigabytes
Gi → Gibibytes
```

For Kubernetes resource configuration, you will commonly see:

```text
CPU    → m
Memory → Mi / Gi
```

Example:

```yaml
resources:
  requests:
    cpu: 200m
    memory: 200Mi

  limits:
    cpu: 500m
    memory: 500Mi
```

---

# 10. ResourceQuota

A **ResourceQuota** limits the total amount of resources that can be consumed by objects within a namespace.

It works at the **namespace level**.

### Why use ResourceQuota?

ResourceQuota helps to:

- Prevent one team from exhausting cluster resources
- Allocate resources fairly between teams
- Prevent excessive resource consumption
- Control the number of objects created in a namespace
- Protect shared cluster resources

Think:

```text
ResourceQuota
      ↓
Namespace-level control
```

---

# 11. ResourceQuota — Compute Resources

Example:

```yaml
apiVersion: v1
kind: ResourceQuota

metadata:
  name: compute-quota

spec:
  hard:
    requests.cpu: "2"
    requests.memory: 4Gi
    limits.cpu: "4"
    limits.memory: 6Gi
```

This quota applies to the namespace containing the ResourceQuota.

It means the namespace can have, in total:

```text
CPU requests  → 2 CPUs
Memory requests → 4Gi

CPU limits → 4 CPUs
Memory limits → 6Gi
```

This is the **combined namespace-level quota**, not a limit on each individual Pod.

---

# 12. ResourceQuota — Object Counts

ResourceQuota can also limit the number of Kubernetes objects in a namespace.

Example:

```yaml
apiVersion: v1
kind: ResourceQuota

metadata:
  name: object-quota

spec:
  hard:
    pods: "10"
    services: "5"
    persistentvolumeclaims: "5"
    secrets: "5"
    configmaps: "6"
    deployments.apps: "10"
```

This limits the number of those objects that can exist in the namespace.

---

# 13. ResourceQuota Example

Imagine a namespace has:

```text
ResourceQuota:
    CPU requests = 2 CPUs
```

And three Pods request:

```text
Pod 1 → 500m
Pod 2 → 500m
Pod 3 → 500m
```

Total:

```text
500m + 500m + 500m
= 1500m
= 1.5 CPU
```

There is still:

```text
2 CPU - 1.5 CPU
= 0.5 CPU
```

available under the quota.

If another Pod requests:

```text
600m
```

the total would become:

```text
2.1 CPU
```

which exceeds the quota.

The Pod creation/admission can therefore be rejected because the namespace would exceed its ResourceQuota.

---

# 14. LimitRange

A **LimitRange** defines default resource requests/limits and constraints for individual Pods or containers within a namespace.

Think:

```text
ResourceQuota
       ↓
Macro level
Namespace

LimitRange
       ↓
Micro level
Container / Pod
```

---

# 15. ResourceQuota vs LimitRange

| Feature | ResourceQuota | LimitRange |
|---|---|---|
| Scope | Namespace | Namespace |
| Controls | Total resource usage | Individual Pod/container defaults and constraints |
| CPU | Yes | Yes |
| Memory | Yes | Yes |
| Object counts | Yes | No |
| Default requests | No | Yes |
| Default limits | No | Yes |
| Main purpose | Control total usage | Control individual resources |

Easy way to remember:

```text
ResourceQuota
    ↓
"How much can the whole namespace consume?"

LimitRange
    ↓
"How much can an individual container/Pod use?"
```

---

# 16. LimitRange Example

Example:

```yaml
apiVersion: v1
kind: LimitRange

metadata:
  name: resource-limits

spec:
  limits:
    - type: Container

      default:
        cpu: 500m
        memory: 500Mi

      defaultRequest:
        cpu: 200m
        memory: 200Mi

      max:
        cpu: "1"
        memory: 1Gi

      min:
        cpu: 100m
        memory: 100Mi
```

This can provide:

```text
Default CPU request → 200m
Default memory request → 200Mi

Default CPU limit → 500m
Default memory limit → 500Mi

Maximum CPU → 1 CPU
Maximum memory → 1Gi

Minimum CPU → 100m
Minimum memory → 100Mi
```

So even if a developer creates a Pod without specifying resources, the LimitRange can apply the configured defaults.

---

# 17. Autoscaling in Kubernetes

**Autoscaling** means automatically adjusting resources based on workload or resource requirements.

For example:

```text
High traffic
    ↓
CPU usage increases
    ↓
More Pods required
    ↓
HPA increases Pod count
```

When demand decreases:

```text
Low traffic
    ↓
CPU usage decreases
    ↓
Fewer Pods required
    ↓
HPA reduces Pod count
```

---

# 18. Types of Kubernetes Autoscaling

There are three major autoscaling concepts:

| Type | Scales | Based On | Scope |
|---|---|---|---|
| HPA | Number of Pods | CPU, Memory, Custom/External Metrics | Pod level |
| VPA | Pod resource requests/limits | Observed resource usage | Pod level |
| Cluster Autoscaler | Number of Nodes | Unschedulable Pods and cluster conditions | Cluster level |

---

# 19. HPA — Horizontal Pod Autoscaler

**HPA** increases or decreases the number of Pod replicas.

Example:

```text
Current:
3 Pods

CPU usage increases
        ↓
HPA
        ↓
5 Pods
```

When traffic decreases:

```text
5 Pods
   ↓
Lower utilization
   ↓
HPA
   ↓
3 Pods
```

---

# 20. HPA Example

Suppose:

```text
Target CPU utilization = 50%
```

If the application is consistently using more CPU than the target:

```text
CPU > 50%
   ↓
HPA increases replicas
```

If utilization falls:

```text
CPU < 50%
   ↓
HPA may reduce replicas
```

HPA makes its decisions using the metrics available to it; it does not simply react instantly to a single CPU spike.

---

# 21. Metrics Server

For CPU and memory resource metrics used by HPA, Kubernetes commonly uses **Metrics Server**.

Check whether Metrics Server is running:

```bash
kubectl get deployment metrics-server -n kube-system
```

You can also check:

```bash
kubectl get pods -n kube-system
```

And test resource metrics:

```bash
kubectl top nodes
```

```bash
kubectl top pods
```

If `kubectl top` does not return metrics, Metrics Server or its configuration may need attention.

---

# 22. Installing Metrics Server

If Metrics Server is not already installed:

```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

Then check:

```bash
kubectl get pods -n kube-system
```

Wait until the Metrics Server Pod is running.

Then:

```bash
kubectl top nodes
```

and:

```bash
kubectl top pods
```

> Managed Kubernetes services such as EKS do not automatically guarantee that Metrics Server is installed. Always verify it in the cluster.

---

# 23. VPA — Vertical Pod Autoscaler

**Vertical Pod Autoscaler** adjusts the CPU and memory resource requests/limits of Pods based on their observed resource usage.

Think:

```text
HPA
 ↓
More / fewer Pods

VPA
 ↓
More / less CPU and memory per Pod
```

Example:

```text
Application needs more CPU
        ↓
VPA
        ↓
Increase recommended/requested CPU
```

VPA is commonly used when workload resource requirements are difficult to determine manually.

VPA requires installing/configuring the VPA components in the cluster.

---

# 24. Cluster Autoscaler

**Cluster Autoscaler (CA)** operates at the node level.

It can adjust the number of nodes in a cluster when additional capacity is needed or when nodes can safely be removed.

Example:

```text
Pod needs 2 CPU
      ↓
No node has enough capacity
      ↓
Pod remains Pending
      ↓
Cluster Autoscaler
      ↓
Adds a node
      ↓
Pod can be scheduled
```

For scale-down, the Cluster Autoscaler can remove nodes when they are underutilized and their workloads can be safely moved elsewhere.

---

# 25. HPA vs VPA vs Cluster Autoscaler

The easiest way to remember:

```text
HPA
 ↓
Horizontal
 ↓
More / fewer Pods


VPA
 ↓
Vertical
 ↓
More / less resources per Pod


Cluster Autoscaler
 ↓
Cluster
 ↓
More / fewer Nodes
```

---

# 26. Generating Load for HPA Testing

A simple load generator can be created using BusyBox:

```bash
kubectl run -i --tty load-generator \
  --rm \
  --image=busybox:1.28 \
  --restart=Never \
  -- /bin/sh -c \
  "while sleep 0.01; do wget -q -O- http://php-apache; done"
```

This continuously sends requests to:

```text
http://php-apache
```

The requests can increase CPU utilization of the target application.

You can then observe:

```bash
kubectl get hpa
```

and:

```bash
kubectl get pods
```

You can also monitor CPU:

```bash
kubectl top pods
```

---

# 27. Resource Management + Autoscaling

These concepts work together.

Example:

```text
                    Kubernetes
                         |
             ┌───────────┴───────────┐
             ↓                       ↓
       Resource Control          Autoscaling
             |                       |
       ┌─────┴─────┐           ┌─────┴─────┐
       ↓           ↓           ↓     ↓     ↓
ResourceQuota  LimitRange     HPA   VPA    CA
       |           |           |     |      |
       ↓           ↓           ↓     ↓      ↓
 Namespace     Pod/Container  Pods  Pod   Nodes
```

---

# 28. Important Points to Remember

### Requests

```text
Minimum resource requirement
```

Used heavily by the scheduler when deciding where a Pod can run.

### Limits

```text
Maximum resource usage allowed
```

### ResourceQuota

```text
Namespace-level total resource/object limit
```

### LimitRange

```text
Individual Pod/container defaults and constraints
```

### HPA

```text
Changes number of Pods
```

### VPA

```text
Adjusts Pod resource requests/limits
```

### Cluster Autoscaler

```text
Changes number of Nodes
```

### Metrics Server

```text
Provides resource usage metrics
```

---

# 29. Final Memory Trick

```text
ResourceQuota
    ↓
"How much can this namespace consume?"

LimitRange
    ↓
"How much can each container/Pod use?"

HPA
    ↓
"How many Pods do I need?"

VPA
    ↓
"How much CPU/Memory does each Pod need?"

Cluster Autoscaler
    ↓
"How many Nodes do I need?"
```

And remember:

```text
CPU      → m
Memory   → Mi / Gi

HPA      → Horizontal → Pods
VPA      → Vertical   → Pod resources
CA       → Cluster    → Nodes
```
