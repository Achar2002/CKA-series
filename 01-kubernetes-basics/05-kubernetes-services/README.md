# 05 - Kubernetes Services

So far, we have learned how to:

- Create Pods
- Access Pods
- Copy files to and from Pods
- Delete Pods
- Check Pod logs
- Execute commands inside Pods

However, Pods are **ephemeral**. Their IP addresses can change when Pods are recreated.

A **Kubernetes Service** provides a stable way to access applications running inside Pods.

---

# 1. What is a Kubernetes Service?

A Kubernetes **Service** is an abstraction that exposes an application running inside Pods.

A Service can provide access:

- Within the Kubernetes cluster
- Outside the Kubernetes cluster

A Pod manifest is used to create Pods.

A Service manifest is used to create a Kubernetes Service.

```text
Client
   |
   v
Service
   |
   v
Pods
```

---

# 2. Types of Kubernetes Services

The commonly used Service types are:

1. **ClusterIP**
2. **NodePort**
3. **LoadBalancer**
4. **ExternalName**
5. **Headless Service**

The first three are especially important for day-to-day Kubernetes and CKA practice.

---

# 3. How Does a Service Know Which Pods to Send Traffic To?

Kubernetes Services use **labels and selectors**.

## Labels

A **label** is a key-value pair attached to a Kubernetes object.

Example:

```yaml
metadata:
  name: nginx-pod
  labels:
    app: nginx-app
    env: dev
    tier: backend
```

Here the Pod has three labels:

```text
app  = nginx-app
env  = dev
tier = backend
```

---

## Selectors

A **selector** is used to select Kubernetes objects based on their labels.

Example:

```yaml
selector:
  app: nginx-app
```

The Service looks for Pods having:

```text
app = nginx-app
```

### Simple way to remember

```text
Label   → attached to the Pod
Selector → used by the Service to find the Pod
```

Example:

```text
              Service
                 |
          selector: app=nginx
                 |
        +--------+--------+
        |        |        |
        v        v        v
      Pod 1    Pod 2    Pod 3
      app=nginx app=nginx app=nginx
```

---

# 4. ClusterIP Service

**ClusterIP** is the default Kubernetes Service type.

It exposes the application **inside the Kubernetes cluster**.

```text
Inside Cluster
      |
      v
  ClusterIP
      |
      v
    Pods
```

The Service receives a stable virtual IP and DNS name that other resources inside the cluster can use.

Example:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-nginx-service
spec:
  type: ClusterIP
  selector:
    app: nginx
  ports:
    - port: 80
      targetPort: 80
```

If `type` is not specified, Kubernetes uses:

```yaml
type: ClusterIP
```

---

## Accessing a ClusterIP Service

Because ClusterIP is available only inside the cluster, we can test it from another Pod.

Example:

```bash
kubectl run -it --rm --image=curlimages/curl --restart=Never -- \
  curl http://my-nginx-service
```

The request flow is:

```text
Temporary curl Pod
       |
       v
my-nginx-service
       |
       v
    Nginx Pod
```

---

# 5. NodePort Service

A **NodePort** Service allows an application to be accessed through the IP address of a Kubernetes node.

The default NodePort range is:

```text
30000 - 32767
```

Example:

```text
NodeIP:NodePort
```

For example:

```text
192.168.1.10:30080
```

---

## NodePort Manifest

Example:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-nginx-service
spec:
  type: NodePort
  selector:
    app: nginx
  ports:
    - port: 80
      targetPort: 80
      nodePort: 30080
```

If `nodePort` is not specified, Kubernetes automatically assigns an available port from the NodePort range.

---

# 6. How NodePort Works

Conceptually:

```text
Client
   |
   v
NodeIP:NodePort
   |
   v
Service
   |
   v
Pod
```

For example:

```text
EC2 Public IP:30080
        |
        v
    NodePort
        |
        v
   Cluster Service
        |
        v
       Pod
```

A NodePort Service also has a ClusterIP internally.

So the NodePort provides an external entry point while the Service provides the normal Kubernetes Service abstraction.

---

# 7. NodePort Traffic Flow

A simplified flow is:

```text
Client
   |
   v
NodeIP:NodePort
   |
   v
Kubernetes Service
   |
   v
Selected Pod
```

The Service selects the appropriate Pods using its selector.

If multiple Pods match the selector, Kubernetes can distribute traffic among the available endpoints.

---

# 8. Important NodePort Consideration

When using NodePort in a cloud environment such as AWS, the required network/security-group/firewall rules must allow traffic to the NodePort.

For example, if the application uses:

```text
NodePort = 30080
```

the relevant security rules must permit the required traffic to port `30080`.

The exact networking behavior depends on the Kubernetes environment and its cloud/network configuration.

---

# 9. LoadBalancer Service

A **LoadBalancer** Service is commonly used when we want to expose an application externally through a cloud provider's load-balancing infrastructure.

Example:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-nginx-service
spec:
  type: LoadBalancer
  selector:
    app: nginx
  ports:
    - port: 80
      targetPort: 80
```

In a supported cloud environment, Kubernetes asks the cloud provider integration to provision external load-balancing infrastructure.

---

# 10. LoadBalancer Traffic Flow

A simplified flow can be represented as:

```text
Internet
   |
   v
Cloud Load Balancer
   |
   v
Kubernetes Service
   |
   v
Pods
```

In some Kubernetes/cloud implementations, NodePort is also involved in the underlying Service implementation.

The exact implementation depends on the cloud provider and Kubernetes configuration.

---

# 11. Service Types - Quick Comparison

| Service Type | Main Purpose | Accessibility |
|---|---|---|
| ClusterIP | Internal application access | Inside cluster |
| NodePort | Basic external access through node IP and port | Outside cluster |
| LoadBalancer | External access using cloud load-balancing infrastructure | Outside cluster |
| ExternalName | Maps a Service name to an external DNS name | DNS-based |
| Headless Service | Service without a ClusterIP | Direct Pod discovery |

---

# 12. How Kubernetes Implements Services

Kubernetes uses networking components to implement Service traffic routing.

Traditionally, **kube-proxy** has been responsible for implementing Service networking rules on nodes.

Depending on the Kubernetes networking implementation, other components such as eBPF-based networking can replace or augment kube-proxy.

For traditional kube-proxy-based networking:

```text
Service
   |
   v
kube-proxy / networking rules
   |
   v
Selected Pod
```

---

# 13. Important Concepts

### Pod

Runs the actual application container.

```text
Pod → Application
```

### Service

Provides a stable way to access the application Pods.

```text
Service → Stable access to Pods
```

### Label

Identifies or categorizes a Kubernetes object.

```text
app=nginx
```

### Selector

Finds objects with matching labels.

```text
selector:
  app: nginx
```

---

# 14. Example Architecture

```text
                    Client
                      |
                      v
              LoadBalancer
                      |
                      v
                 Service
                      |
             selector: app=nginx
                      |
          +-----------+-----------+
          |           |           |
          v           v           v
       Pod 1        Pod 2       Pod 3
       nginx        nginx       nginx
```

The important relationship is:

```text
Service Selector
       ↓
Pod Labels
       ↓
Matching Pods
```

---

# 15. Useful Commands

List Services:

```bash
kubectl get services
```

or:

```bash
kubectl get svc
```

Get detailed Service information:

```bash
kubectl describe service <service-name>
```

View Services with their ClusterIP and ports:

```bash
kubectl get svc
```

View Pods and their labels:

```bash
kubectl get pods --show-labels
```

View a specific Pod's labels:

```bash
kubectl get pod <pod-name> --show-labels
```

Check Service endpoints:

```bash
kubectl get endpoints
```

For newer Kubernetes versions, EndpointSlices are also available:

```bash
kubectl get endpointslices
```

---

# 16. Key Takeaways

- Pods are ephemeral, so their IP addresses can change.
- Services provide a stable way to access Pods.
- A Service uses **selectors** to identify matching Pods.
- Pods use **labels** to identify themselves.
- ClusterIP is the default Service type.
- ClusterIP provides internal cluster access.
- NodePort exposes a Service through a port on Kubernetes nodes.
- NodePort ports normally come from the `30000-32767` range.
- LoadBalancer can provision external cloud load-balancing infrastructure when supported.
- ExternalName provides DNS-based mapping to an external name.
- Headless Services do not have a ClusterIP and are useful for direct Pod discovery.
- kube-proxy traditionally implements Service networking using node-level networking rules.
- Services provide stable access even when individual Pods are replaced.

---

## What I Practiced

- Understanding Kubernetes Services
- Understanding labels and selectors
- Creating ClusterIP Services
- Accessing applications through ClusterIP
- Understanding NodePort
- Understanding the NodePort range
- Understanding LoadBalancer Services
- Understanding Service traffic flow
- Understanding how Services identify Pods using selectors
- Checking Services and Service endpoints
