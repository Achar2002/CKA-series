# 14. Kubernetes Probes and Cluster Autoscaler

## Part 1: Kubernetes Probes

### 1. What Are Kubernetes Probes?

In Kubernetes, **probes are health checks performed by the kubelet** to determine whether a container is starting correctly, running properly, and ready to receive traffic.

A Pod can have a `Running` status even when the application inside it is not ready to serve requests or is stuck.

Probes help Kubernetes respond to these situations automatically.

### 2. Why Do We Need Probes?

Consider an e-commerce application running on Amazon EKS:

- The application takes 30 seconds to establish database connections.
- The application occasionally hangs because of a bug or resource issue.
- Kubernetes reports the Pod as `Running`, but the application may not work correctly.

**Without probes:**
- Traffic may reach the application before it is ready.
- A hung application may continue running without being restarted.

**With probes:**
- Readiness probes control when the Pod can receive Service traffic.
- Liveness probes restart containers that repeatedly fail their health checks.
- Startup probes protect applications that need extra time to initialize.

### 3. Types of Kubernetes Probes

| Probe | What does it check? | What happens when it fails? |
|---|---|---|
| Liveness | Is the application healthy enough to keep running? | Kubernetes restarts the container after the configured failures. |
| Readiness | Can the application handle requests now? | The Pod is marked not ready and normally stops receiving traffic through matching Services. |
| Startup | Has the application completed startup? | The container is restarted if the startup probe exceeds its failure threshold. Liveness and readiness checks are held back until startup succeeds. |

**Important:** A failed readiness probe does not restart the container. A failed liveness or startup probe can cause a container restart.

### 4. Liveness Probe

A liveness probe detects an application that is running but has become unresponsive or stuck.

If the probe fails repeatedly, Kubernetes restarts the container.

Example:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: liveness-demo
spec:
  containers:
    - name: nginx
      image: nginx:1.28
      ports:
        - containerPort: 80
      livenessProbe:
        httpGet:
          path: /
          port: 80
        initialDelaySeconds: 5
        periodSeconds: 10
        timeoutSeconds: 2
        failureThreshold: 3
```

Save this as `liveness-pod.yaml`.

Apply and inspect it:

```bash
kubectl apply -f liveness-pod.yaml
kubectl get pod liveness-demo
kubectl describe pod liveness-demo
```

The probe sends an HTTP request to `/` on port 80. If it fails three consecutive times, Kubernetes restarts the container.

### 5. Readiness Probe

A readiness probe determines whether a container is ready to accept traffic.

If readiness fails, the Pod remains running, but it is marked `NotReady`. Kubernetes removes it from the set of ready endpoints used by normal Service traffic routing.

Example:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: readiness-demo
  labels:
    app: readiness-demo
spec:
  containers:
    - name: nginx
      image: nginx:1.28
      ports:
        - containerPort: 80
      readinessProbe:
        httpGet:
          path: /
          port: 80
        initialDelaySeconds: 5
        periodSeconds: 5
        timeoutSeconds: 2
        failureThreshold: 3
---
apiVersion: v1
kind: Service
metadata:
  name: readiness-service
spec:
  selector:
    app: readiness-demo
  ports:
    - port: 80
      targetPort: 80
```

Save as `readiness-pod.yaml`.

```bash
kubectl apply -f readiness-pod.yaml
kubectl get pods
kubectl get svc readiness-service
kubectl get endpoints readiness-service
```

The endpoint list shows which Pod IPs are available to receive Service traffic. On newer Kubernetes versions, you can also inspect EndpointSlices:

```bash
kubectl get endpointslices \
  -l kubernetes.io/service-name=readiness-service
```

### 6. Startup Probe

A startup probe is useful for applications that need a long time to initialize.

It prevents liveness and readiness probes from interfering with startup before the application is ready.

Example:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: startup-demo
spec:
  containers:
    - name: nginx
      image: nginx:1.28
      ports:
        - containerPort: 80
      startupProbe:
        httpGet:
          path: /
          port: 80
        periodSeconds: 5
        failureThreshold: 12
      livenessProbe:
        httpGet:
          path: /
          port: 80
        periodSeconds: 10
        failureThreshold: 3
      readinessProbe:
        httpGet:
          path: /
          port: 80
        periodSeconds: 5
        failureThreshold: 3
```

Save as `startup-pod.yaml`.

```bash
kubectl apply -f startup-pod.yaml
kubectl describe pod startup-demo
```

Here, the startup probe allows approximately 60 seconds for startup failures before the container is restarted (`12 × 5 seconds`, approximately). Probe scheduling and timeouts can affect the exact elapsed time.

Once the startup probe succeeds, liveness and readiness checks can begin.

### 7. Probe Configuration Parameters

| Parameter | Meaning |
|---|---|
| `initialDelaySeconds` | How long to wait after container startup before beginning the probe. |
| `periodSeconds` | How often to run the probe. |
| `timeoutSeconds` | How long to wait for a probe result before treating it as a failure. |
| `successThreshold` | Number of consecutive successes required to consider the probe successful. |
| `failureThreshold` | Number of consecutive failures required before the probe is considered failed. |

Notes:
- `periodSeconds` is the correct parameter name, not `periodSecond`.
- `successThreshold` must be `1` for startup and liveness probes.
- Readiness probes can use a higher success threshold.
- Defaults apply when you omit optional settings.

### 8. Probe Mechanisms

Kubernetes supports several ways to check container health:

- **HTTP GET:** Requests an HTTP endpoint and checks the response status.
- **TCP Socket:** Checks whether a TCP connection can be established.
- **Exec:** Runs a command inside the container and checks its exit status.
- **gRPC:** Checks application health using the gRPC health-check protocol.

Example of an `exec` probe:

```yaml
livenessProbe:
  exec:
    command:
      - cat
      - /tmp/healthy
  initialDelaySeconds: 5
  periodSeconds: 10
```

The command succeeds if `/tmp/healthy` exists and can be read. The application or another process must create and maintain this file appropriately.

### 9. Termination Grace Period

When Kubernetes needs to stop a container, it normally sends `SIGTERM` and waits for the configured termination grace period before forcefully terminating the process.

Example:

```yaml
spec:
  terminationGracePeriodSeconds: 30
```

This gives the application time to stop accepting new work, finish or safely abandon in-flight requests, and close connections.

**Important:** The grace period does not automatically save user data. The application must implement graceful shutdown and persist its data correctly.

### 10. Interview Quick Revision

- **Liveness:** Should this container be restarted?
- **Readiness:** Should this Pod receive traffic?
- **Startup:** Has this application finished starting?
- **Pod Running vs Ready:** Running indicates the Pod has been scheduled and its containers have started; Ready indicates it currently satisfies the readiness conditions.
- **Readiness failure:** Does not restart the container.
- **Liveness failure:** Can trigger a container restart.
- **Startup probe:** Prevents liveness/readiness checks from acting before startup succeeds.

---

## Part 2: Kubernetes Cluster Autoscaler on AWS EKS

### 1. What Is Cluster Autoscaler?

The **Cluster Autoscaler** adjusts the number of cluster nodes when capacity needs change.

It helps Kubernetes workloads obtain sufficient node resources and can remove suitable underutilized nodes.

| Situation | Cluster Autoscaler behavior |
|---|---|
| Pods cannot be scheduled because of insufficient node resources | Can add nodes if a configured node group can scale up and the new nodes can accommodate the Pods. |
| Nodes are underutilized and their workloads can be moved elsewhere | Can remove eligible nodes. |

**Important distinction:**

- **HPA (Horizontal Pod Autoscaler):** Adjusts the number of Pod replicas.
- **Cluster Autoscaler:** Adjusts the number of cluster nodes.

The Cluster Autoscaler does not simply add a node whenever CPU usage is high. A common scale-up trigger is a Pod that cannot be scheduled because no suitable node has enough resources.

### 2. Prerequisites for EKS

Before installing Cluster Autoscaler, ensure that:

1. Your EKS cluster exists.
2. A supported node group or Auto Scaling Group is configured for scaling.
3. The Cluster Autoscaler has the required AWS IAM permissions.
4. Its AWS region and cluster discovery settings are correct.
5. The Auto Scaling Group's minimum, maximum, and desired capacities are configured appropriately.

For EKS, grant permissions using an appropriate IAM role, commonly through EKS Pod Identity or IAM Roles for Service Accounts (IRSA). Follow the AWS and Cluster Autoscaler documentation for the method you use.

Do not assume installing the Helm chart alone configures AWS permissions or node-group discovery.

### 3. Install Helm (if required)

Check whether Helm is installed:

```bash
helm version
```

If it is not installed, use the official Helm installation instructions for your operating system. Review installation scripts before running them with elevated privileges.

### 4. Add the Cluster Autoscaler Helm Repository

```bash
helm repo add autoscaler https://kubernetes.github.io/autoscaler
helm repo update
```

Inspect the chart options before installing:

```bash
helm show values autoscaler/cluster-autoscaler
```

### 5. Configure Cluster Autoscaler for EKS

The exact values depend on your cluster name, region, IAM setup, and chart version. The following is an illustrative installation pattern:

```bash
helm install cluster-autoscaler autoscaler/cluster-autoscaler \
  --namespace kube-system \
  --set autoDiscovery.clusterName=<cluster-name> \
  --set awsRegion=<region-code> \
  --set extraArgs.balance-similar-node-groups=true
```

Replace `<cluster-name>` and `<region-code>` with your actual values. Do not type the angle brackets literally.

For example, if your cluster is `kastro-cluster` in `us-east-1`, use those values.

The node group's Auto Scaling Group must be discoverable using the tags and permissions expected by Cluster Autoscaler. Verify the current chart's values and AWS setup before deploying.

Avoid setting `skip-nodes-with-system-pods=false` without understanding the consequences. It changes the protection around removing nodes that host system Pods and is not a universal requirement.

### 6. Verify the Installation

```bash
kubectl get pods -n kube-system
kubectl logs -n kube-system deployment/cluster-autoscaler
kubectl describe deployment cluster-autoscaler -n kube-system
```

Look for errors involving IAM permissions, Auto Scaling Group discovery, AWS region, or cluster name.

### 7. Deploy a PHP-Apache Test Application

Create `php-apache.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: php-apache
spec:
  replicas: 1
  selector:
    matchLabels:
      run: php-apache
  template:
    metadata:
      labels:
        run: php-apache
    spec:
      containers:
        - name: php-apache
          image: registry.k8s.io/hpa-example
          ports:
            - containerPort: 80
          resources:
            requests:
              cpu: 200m
            limits:
              cpu: 500m
---
apiVersion: v1
kind: Service
metadata:
  name: php-apache
spec:
  selector:
    run: php-apache
  ports:
    - port: 80
      targetPort: 80
```

Apply the manifest:

```bash
kubectl apply -f php-apache.yaml
kubectl get deployment php-apache
kubectl get pods -o wide
kubectl get svc php-apache
```

The Service provides network access to the Pods, while the Deployment manages their replicas.

### 8. Scale Pods to Test Node Autoscaling

First, inspect current capacity:

```bash
kubectl get nodes
kubectl get pods -o wide
kubectl describe nodes
```

Increase the number of replicas:

```bash
kubectl scale deployment php-apache --replicas=10
kubectl get pods -o wide
kubectl get events --sort-by=.lastTimestamp
```

If the cluster lacks enough capacity, some Pods may remain `Pending`. If the node group is correctly configured and can scale, Cluster Autoscaler may request additional nodes.

**Note:** Ten replicas may fit on the existing nodes. If so, no node scale-up will occur. A meaningful test depends on the resource requests, available capacity, node-group limits, and scheduling constraints.

### 9. How to Trigger Scale-Up Reliably

For a controlled lab, use a test workload with resource requests large enough that the existing nodes cannot fit all replicas. Make sure the configured node type can accommodate an individual Pod.

Monitor the workload:

```bash
kubectl get pods -w
```

In another terminal:

```bash
kubectl get nodes -w
```

Check Cluster Autoscaler logs:

```bash
kubectl logs -n kube-system deployment/cluster-autoscaler --follow
```

If Pods stay pending, inspect the reason:

```bash
kubectl describe pod <pending-pod-name>
```

Look for messages such as `Insufficient cpu` or `Insufficient memory`. Also check Cluster Autoscaler logs for node-group discovery, maximum-size, or IAM errors.

### 10. Scale-Down Behavior

Cluster Autoscaler can remove eligible nodes when it determines their workloads can safely run elsewhere.

Scale-down is not immediate. It depends on utilization, Pod scheduling constraints, disruption rules, local storage, system Pods, and the Cluster Autoscaler configuration.

Before testing scale-down, confirm that the workload can be moved safely and that you are not removing the only node capable of running an important workload.

### 11. Clean Up the Test Application

Remove the test workload:

```bash
kubectl delete -f php-apache.yaml
```

If you installed Cluster Autoscaler only for this lab and no workloads depend on it, uninstall the Helm release:

```bash
helm uninstall cluster-autoscaler -n kube-system
```

Uninstalling the chart does not necessarily undo IAM roles, policies, or Auto Scaling Group tags you created. Review and remove lab-specific AWS resources separately.

### 12. Interview Quick Revision

- **HPA:** Scales Pods.
- **Cluster Autoscaler:** Scales nodes.
- **Pending Pods:** Can trigger scale-up when no existing node can accommodate them and a suitable node group can grow.
- **Scale-down:** Removes eligible nodes after workloads can be moved safely.
- **Helm:** Simplifies installation and management of Kubernetes applications.
- **AWS IAM:** Cluster Autoscaler needs permission to discover and adjust the relevant Auto Scaling Groups.
- **Node group maximum size:** Limits how many nodes Cluster Autoscaler can add.

### Key Takeaway

Kubernetes probes help keep applications healthy and route traffic only to ready Pods. Cluster Autoscaler helps provide node capacity when workloads cannot fit on the existing nodes. Together, they support more reliable workloads on Kubernetes and EKS.
