# 04 - Pod Practical Commands and Init Containers

This section covers hands-on Pod operations and practical examples of Init Containers.

---

# 1. Checking Pod Resource Usage

To view CPU and memory usage of Pods:

```bash
kubectl top pod
```

To check resource usage of a specific Pod:

```bash
kubectl top pod single-container-pod
```

> `kubectl top` requires the Metrics Server to be available in the cluster.

---

# 2. Working with Files in a Pod

Kubernetes provides `kubectl cp` to copy files between the local machine and a Pod.

## Copy a file from the local machine into a Pod

Create a file:

```bash
touch kastro.txt
```

Check the current directory:

```bash
pwd
```

Copy the file into the Pod:

```bash
kubectl cp /root/kastro.txt single-container-pod:/tmp/
```

The file is now available inside:

```text
/tmp/kastro.txt
```

---

# 3. Entering a Pod

To open an interactive shell inside a container:

```bash
kubectl exec -it single-container-pod -- /bin/bash
```

This allows us to execute commands directly inside the container.

For images that do not contain Bash, such as some Alpine-based images, use:

```bash
kubectl exec -it <pod-name> -- /bin/sh
```

---

# 4. Copying a File from Pod to Local Machine

To copy a file from a Pod to the local machine:

```bash
kubectl cp single-container-pod:/tmp/k8s.txt /root/
```

You can also specify the destination file:

```bash
kubectl cp single-container-pod:/tmp/k8s.txt /root/k8s.txt
```

Verify the file:

```bash
ls
```

---

# 5. Checking Pod Status

List Pods:

```bash
kubectl get pods
```

For additional information such as the Pod IP and node:

```bash
kubectl get pods -o wide
```

---

# 6. Viewing Container Logs

To view the logs of a Pod:

```bash
kubectl logs single-container-pod
```

If the Pod contains multiple containers, specify the container:

```bash
kubectl logs single-container-pod -c nginx-sc-container
```

The `-c` option identifies which container's logs should be displayed.

---

# 7. Init Containers

An **Init Container** is a special container that runs before the main application containers in a Pod.

The main application containers start only after all Init Containers complete successfully.

Basic flow:

```text
Pod Created
     |
     v
Init Container
     |
     v
Init Container completes
     |
     v
Application Container starts
```

Init Containers are useful for:

* Initialization tasks
* Preparing configuration files
* Waiting for dependencies
* Performing setup before the application starts

---

# 8. Init Container Example 1 - Basic Concept

A Pod can contain:

```text
Pod
│
├── Init Container
│       ↓
│   completes
│       ↓
└── Application Container
```

The Init Container runs first.

If an Init Container fails, Kubernetes restarts it according to the Pod's restart policy and the application container does not start until the Init Container completes successfully.

---

# 9. Example 2 - Init Container Waiting for a Service

## Scenario

Imagine that we want Nginx to start only after a backend service becomes available.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: init-wait-for-service
spec:
  initContainers:
    - name: wait-for-backend
      image: busybox
      command: ['sh', '-c']
      args:
        - |
          until nslookup my-backend-service.default.svc.cluster.local; do
            echo "Waiting for backend..."
            sleep 5
          done

  containers:
    - name: nginx
      image: nginx:alpine
      ports:
        - containerPort: 80
```

### How it works

The Init Container runs:

```bash
nslookup my-backend-service.default.svc.cluster.local
```

If the Service cannot be resolved, the loop continues:

```text
Waiting for backend...
Waiting for backend...
Waiting for backend...
```

Once the Service becomes resolvable:

```text
Init Container completes
        ↓
Nginx container starts
```

### Important

If the Service does not exist, the Pod remains in the **Init** phase because the Init Container never completes.

---

## Practical Commands

Apply the manifest:

```bash
kubectl apply -f init-wait-for-service.yml
```

Check the Pod:

```bash
kubectl get pod init-wait-for-service
```

Describe the Pod:

```bash
kubectl describe pod init-wait-for-service
```

Check Init Container logs:

```bash
kubectl logs init-wait-for-service -c wait-for-backend
```

The logs help us understand why the Pod is waiting.

---

# 10. Example 3 - Init Container Writing a Configuration File

## Scenario

The Init Container creates a file, and the Nginx container serves that file.

The two containers communicate through a shared `emptyDir` volume.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: init-write-config
spec:
  volumes:
    - name: config-volume
      emptyDir: {}

  initContainers:
    - name: init-config
      image: busybox
      command: ['sh', '-c']
      args:
        - |
          echo "Hello from init container" > /config/index.html

      volumeMounts:
        - name: config-volume
          mountPath: /config

  containers:
    - name: nginx
      image: nginx:alpine
      ports:
        - containerPort: 80

      volumeMounts:
        - name: config-volume
          mountPath: /usr/share/nginx/html
```

---

# 11. How Example 3 Works

The flow is:

```text
                Shared emptyDir Volume
                       |
        +--------------+--------------+
        |                             |
        v                             v
 Init Container                  Nginx Container
        |                             |
 Writes index.html              Serves index.html
        |                             |
 /config/index.html        /usr/share/nginx/html
```

### Step 1 - Init Container starts

The Init Container executes:

```bash
echo "Hello from init container" > /config/index.html
```

### Step 2 - File is stored in the shared volume

The file is written to:

```text
/config/index.html
```

### Step 3 - Nginx starts

The same volume is mounted inside Nginx at:

```text
/usr/share/nginx/html
```

Therefore, Nginx can access:

```text
/usr/share/nginx/html/index.html
```

### Step 4 - Nginx serves the file

When we access Nginx, the response is:

```text
Hello from init container
```

---

# 12. Practical Commands for Example 3

Apply the manifest:

```bash
kubectl apply -f init-write-config.yml
```

Check the Pod:

```bash
kubectl get pod init-write-config
```

Wait until the Pod reaches `Running`:

```bash
kubectl get pod init-write-config
```

Port-forward the Pod:

```bash
kubectl port-forward pod/init-write-config 8080:80
```

Then access:

```bash
curl http://localhost:8080
```

Expected output:

```text
Hello from init container
```

---

# 13. Important Commands Practiced

| Command                    | Purpose                                   |
| -------------------------- | ----------------------------------------- |
| `kubectl top pod`          | View Pod resource usage                   |
| `kubectl get pods`         | List Pods                                 |
| `kubectl get pods -o wide` | View additional Pod information           |
| `kubectl exec -it`         | Execute commands inside a container       |
| `kubectl cp`               | Copy files between local machine and Pod  |
| `kubectl logs`             | View container logs                       |
| `kubectl describe pod`     | View detailed Pod information             |
| `kubectl apply -f`         | Create/update resources from a manifest   |
| `kubectl port-forward`     | Temporarily forward a local port to a Pod |

---

# 14. Key Takeaways

* `kubectl top pod` can be used to view Pod CPU and memory usage when Metrics Server is available.
* `kubectl exec` allows us to execute commands inside containers.
* `kubectl cp` transfers files between the local machine and a Pod.
* `kubectl logs` is used to inspect container logs.
* Init Containers always run before the main application containers.
* The application containers start only after all Init Containers successfully complete.
* Init Containers can be used to wait for dependencies.
* Init Containers can prepare configuration or files required by the application.
* `emptyDir` can be used as shared temporary storage between containers in the same Pod.
* Pod networking and storage can be shared between containers within the same Pod.

---

## What I Practiced

* Checking Pod resource usage
* Executing commands inside Pods
* Copying files into and out of Pods
* Viewing container logs
* Creating and troubleshooting Init Containers
* Using Init Containers to wait for a dependency
* Sharing files between Init and application containers using `emptyDir`
* Serving an Init Container-generated file through Nginx
