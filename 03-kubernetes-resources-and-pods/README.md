# 03 - Understanding Kubernetes Resources and Pods

## 1. Understanding Kubernetes Resources

In Kubernetes, everything we create or manage is represented as a **resource (object)**.

Examples:

* Pods
* Deployments
* StatefulSets
* DaemonSets
* Services
* ConfigMaps
* Secrets

Kubernetes resources generally have:

* **Desired State** – what we want the resource to look like.
* **Current/Observed State** – what is actually running in the cluster.

Kubernetes continuously works to make the current state match the desired state.

### View available Kubernetes resources

```bash
kubectl api-resources
```

This command lists the resource types supported by the Kubernetes API server.

---

# 2. Methods of Creating Kubernetes Resources

There are two common approaches.

## 2.1 Imperative Approach

Resources are created directly using commands.

Example:

```bash
kubectl run nginx --image=nginx
```

### Characteristics

* Quick and simple
* Useful for testing and learning
* Commands are not as reusable
* Difficult to maintain for larger environments

---

## 2.2 Declarative Approach

Resources are defined using **manifest files** and applied to the cluster.

Example:

```bash
kubectl apply -f pod.yaml
```

### Characteristics

* Configuration is stored as code
* Reusable
* Easier to maintain
* Suitable for automation and CI/CD
* Commonly preferred for production workloads

---

# 3. What is a Manifest File?

A **manifest file** is a configuration file that describes a Kubernetes resource and its desired state.

It tells Kubernetes **what we want to create and how it should be configured**.

Example:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
spec:
  containers:
    - name: nginx
      image: nginx
```

The manifest can then be applied using:

```bash
kubectl apply -f pod.yaml
```

---

# 4. Why Use Manifest Files?

Manifest files provide several benefits:

### Automation

They can be used in CI/CD pipelines and automated deployments.

### Clarity

The desired configuration is clearly written in a file.

### Reusability

The same manifest can be reused across environments with appropriate configuration changes.

### Version Control

Manifest files can be stored in Git and tracked like application source code.

---

# 5. Manifest File Formats

Kubernetes manifests are commonly written in:

### YAML

**YAML** is the most commonly used format for Kubernetes manifests.

File extensions:

```text
.yaml
.yml
```

### JSON

Kubernetes also supports JSON manifests.

```text
.json
```

YAML is generally preferred because it is easier for humans to read and maintain.

---

# 6. Structure of a Kubernetes Manifest

A typical Kubernetes manifest contains four important sections:

```yaml
apiVersion:
kind:
metadata:
spec:
```

## 6.1 apiVersion

Defines the Kubernetes API version used by the resource.

Example:

```yaml
apiVersion: v1
```

---

## 6.2 kind

Defines the type of Kubernetes resource we want to create.

Examples:

```yaml
kind: Pod
```

```yaml
kind: Deployment
```

```yaml
kind: Service
```

---

## 6.3 metadata

Contains identifying information about the resource.

Example:

```yaml
metadata:
  name: nginx-pod
  labels:
    app: nginx
```

---

## 6.4 spec

Defines the desired configuration of the resource.

For a Pod, this can include:

* Container name
* Container image
* Ports
* Volumes
* Environment variables
* Resource requirements

Example:

```yaml
spec:
  containers:
    - name: nginx
      image: nginx
      ports:
        - containerPort: 80
```

> **Important:** YAML indentation is significant. Incorrect indentation can cause the manifest to fail.

YAML is also **case-sensitive**.

---

# 7. What is a Pod?

A **Pod is the smallest deployable unit in Kubernetes**.

A Pod provides the environment in which one or more containers run.

A Pod can contain:

* One container
* Multiple containers

Containers within the same Pod share:

* Network namespace
* Storage volumes
* Pod lifecycle

For most application workloads, having **one main application container per Pod** is the common approach.

---

# 8. Pod Example

A simple Pod manifest:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
spec:
  containers:
    - name: nginx
      image: nginx
```

Create the Pod:

```bash
kubectl apply -f pod.yaml
```

Check the Pod:

```bash
kubectl get pods
```

Get more information:

```bash
kubectl describe pod nginx-pod
```

---

# 9. Pod Lifecycle

At a high level, Pods can move through these phases:

```text
Pending
   |
   v
Running
   |
   +----> Succeeded
   |
   +----> Failed
```

There is also an **Unknown** phase when the Kubernetes control plane cannot obtain the Pod's state.

### Pod phases

| Phase     | Meaning                                                               |
| --------- | --------------------------------------------------------------------- |
| Pending   | Pod has been accepted but is not yet running                          |
| Running   | Pod has been assigned to a node and at least one container is running |
| Succeeded | All containers completed successfully                                 |
| Failed    | All containers terminated and at least one failed                     |
| Unknown   | Kubernetes cannot determine the Pod's current state                   |

### Container states

Individual containers have their own states:

```text
Waiting → Running → Terminated
```

---

# 10. Types of Pods

## 10.1 Single-Container Pod

The most common basic pattern:

```text
Pod
└── Container
```

One Pod contains one main application container.

---

## 10.2 Multi-Container Pod / Sidecar Pattern

A Pod can contain multiple containers:

```text
Pod
├── Application Container
└── Sidecar Container
```

The containers share the Pod's network and can share volumes.

### Example use case

An application container generates logs while a sidecar container processes or forwards those logs.

Other sidecar use cases include:

* Log collection
* Proxying
* Supporting application functionality

---

## 10.3 Init Container

An **init container** runs before the main application containers start.

Example:

```text
Pod
│
├── Init Container
│       ↓
│   completes
│       ↓
├── Application Container
└── Sidecar Container
```

Init containers are useful for tasks such as:

* Preparing files
* Waiting for a dependency
* Performing initialization tasks

---

## 10.4 Static Pod

A **static Pod** is managed directly by the **kubelet** on a node rather than through the Kubernetes API server in the normal way.

Static Pods are commonly used for certain control-plane components in self-managed Kubernetes clusters.

---

# 11. How Pods Communicate

Kubernetes provides a networking model where Pods receive their own IP addresses.

Consider two Pods:

```text
Pod A                    Pod B
10.244.1.10  ─────────>  10.244.1.11
```

Pods can communicate with each other using their Pod IP addresses.

However, Pod IPs are **ephemeral**. If a Pod is deleted and recreated, the replacement Pod may receive a different IP address.

This is one reason applications normally use a **Service** instead of directly depending on Pod IPs.

---

# 12. Communication Within the Same Pod

Containers inside the same Pod share the same network namespace.

Therefore, they share:

* The same Pod IP
* The same network interface
* The same localhost

For example:

```text
Pod
├── Application Container
│       └── localhost:8080
│
└── Sidecar Container
        └── localhost:8080
```

The containers can communicate with each other using `localhost` and the appropriate port.

---

# 13. Communication Between Pods on the Same Node

Each Pod has its own IP address.

Example:

```text
Node
│
├── Pod A → 10.244.1.10
│
└── Pod B → 10.244.1.11
```

Pod A can communicate with Pod B using Pod B's IP.

The Kubernetes networking model is designed so that Pods can communicate without requiring NAT between Pods.

---

# 14. Communication Between Pods on Different Nodes

Pods can also communicate across nodes.

Example:

```text
Node 1                         Node 2

Pod A                          Pod B
10.244.1.10  ───────────────> 10.244.2.10
```

Kubernetes networking requires Pod-to-Pod communication across the cluster without requiring NAT for Pod-to-Pod traffic.

The actual networking implementation is provided by the cluster's **CNI plugin**.

---

# 15. What is a Pod IP?

A **Pod IP** is the IP address assigned to a Pod within the cluster network.

It can be used for:

* Pod-to-Pod communication
* Communication from other cluster components to the Pod

However, Pod IPs are temporary because Pods are ephemeral.

For stable access to applications, Kubernetes **Services** provide a stable virtual IP and DNS name.

---

# 16. How Are Pod IPs Allocated?

Kubernetes networking uses a **CNI (Container Network Interface) plugin** to implement Pod networking.

A simplified flow is:

```text
Pod is scheduled
       ↓
Kubelet creates the Pod sandbox
       ↓
Kubelet invokes the CNI plugin
       ↓
CNI configures the Pod network
       ↓
Pod receives an IP address
```

The exact IP allocation mechanism depends on the CNI implementation.

In many cluster configurations, each node is associated with a Pod IP range (Pod CIDR), and the networking system allocates Pod addresses from the appropriate range.

---

# 17. Important Commands

### List available Kubernetes resources

```bash
kubectl api-resources
```

### Create a resource from a manifest

```bash
kubectl apply -f pod.yaml
```

### List Pods

```bash
kubectl get pods
```

### Get detailed Pod information

```bash
kubectl describe pod <pod-name>
```

### View Pod IP and node information

```bash
kubectl get pods -o wide
```

### Delete a Pod

```bash
kubectl delete pod <pod-name>
```

---

# 18. Key Takeaways

* Kubernetes manages resources using a desired-state model.
* Resources can be created imperatively or declaratively.
* Manifest files define the desired configuration of Kubernetes resources.
* YAML is the most commonly used manifest format.
* A manifest commonly contains `apiVersion`, `kind`, `metadata`, and `spec`.
* A Pod is the smallest deployable unit in Kubernetes.
* A Pod can contain one or multiple containers.
* Containers in the same Pod share the network namespace.
* Pods receive their own IP addresses.
* Pod IPs are ephemeral.
* Kubernetes networking allows Pod-to-Pod communication across nodes.
* CNI plugins implement Pod networking and IP allocation.
* Services are used when stable network access to Pods is required.

---

## What I Practiced

* Understanding Kubernetes resources
* Using `kubectl api-resources`
* Imperative vs declarative resource creation
* Understanding Kubernetes manifest files
* Understanding the structure of YAML manifests
* Creating and inspecting Pods
* Understanding Pod lifecycle and container states
* Understanding single-container, multi-container, init, and static Pods
* Understanding Pod networking and Pod IP allocation
