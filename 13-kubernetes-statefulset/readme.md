# Kubernetes StatefulSet — Theory and Practical Demos

## 1. What Is a StatefulSet?

A **StatefulSet** is a Kubernetes workload controller designed for applications that need stable identity, predictable Pod names, and persistent storage.

Examples include:

- MySQL
- PostgreSQL
- MongoDB
- Kafka
- Stateful distributed applications

Unlike a Deployment, a StatefulSet gives its Pods stable identities.

### Deployment vs StatefulSet

A Deployment might create Pods with names like:

```text
mysql-deployment-66b6d79c8-sb98s
mysql-deployment-66b6d79c8-mptvs
mysql-deployment-66b6d79c8-x7k2p
```

The suffixes are generated dynamically. If a Pod is deleted, its replacement generally receives a different name.

A StatefulSet creates Pods with predictable names:

```text
mysql-0
mysql-1
mysql-2
```

If `mysql-1` is deleted, Kubernetes recreates it with the same name: `mysql-1`.

### Important features of StatefulSet

- **Stable identity:** predictable Pod names.
- **Stable network identity:** individual Pod DNS names when configured with a Headless Service.
- **Persistent storage:** can use PVCs to preserve data beyond Pod recreation.
- **Ordered deployment and scaling:** Pods are normally created and removed in an ordered sequence.
- **Ordered rolling updates:** updates follow StatefulSet-specific rollout behavior.

**Important:** StatefulSet does not automatically make an application highly available or replicate its database data. The application must support replication or clustering when those features are required.

---

## 2. Why Not Use a Deployment for Every Application?

Consider three Nginx Pods managed by a Deployment:

```text
Deployment
    |
    +-- nginx-7c9d-abc12
    +-- nginx-7c9d-def34
    +-- nginx-7c9d-ghi56
```

These Pods are interchangeable. If one fails, the Deployment creates a replacement.

This works well for stateless applications such as web servers.

However, consider a database:

```text
MySQL Pod
    |
    +-- Database
    +-- Tables
    +-- Records
```

If the database stores its data only in the container filesystem or a Pod-scoped `emptyDir`, deleting the Pod can result in data loss.

Even with a Deployment and persistent storage, multiple database replicas do not automatically share or replicate the same database contents.

For stateful applications, we need to plan both **workload identity and storage**.

---

## 3. PV/PVC vs StatefulSet

| Feature | PV/PVC | StatefulSet |
|---|---|---|
| Purpose | Storage management | Workload management |
| Provides persistent storage | Yes, through the underlying storage | No, not by itself |
| Stable Pod identity | No | Yes |
| Manages replicas | No | Yes |
| Creates per-Pod PVCs automatically | No | Can, using `volumeClaimTemplates` |
| Can be used together | Yes | Yes |

A StatefulSet often uses PVs and PVCs to preserve application data.

---

## 4. Components of a StatefulSet

### A. StatefulSet manifest

Defines the application, replica count, Pod template, and storage configuration.

### B. Headless Service

A Headless Service provides stable DNS records for individual StatefulSet Pods.

Example:

```text
mysql-0.mysql-service.default.svc.cluster.local
mysql-1.mysql-service.default.svc.cluster.local
mysql-2.mysql-service.default.svc.cluster.local
```

These names assume:

- StatefulSet name: `mysql`
- Headless Service name: `mysql-service`
- Namespace: `default`

### What is a Headless Service?

A Headless Service is a Service configured with:

```yaml
clusterIP: None
```

Unlike a regular Service, it does not provide a single cluster IP for load balancing. Instead, Kubernetes DNS can resolve the Service to its backing Pod addresses and provide individual Pod DNS records.

### C. `volumeClaimTemplates`

A StatefulSet can use `volumeClaimTemplates` to create a separate PVC for each Pod.

For example:

```text
mysql-0 → mysql-storage-mysql-0
mysql-1 → mysql-storage-mysql-1
mysql-2 → mysql-storage-mysql-2
```

This is useful when each database replica needs its own persistent data directory.

### D. StorageClass and PV

A StorageClass can dynamically provision storage, while PVs represent the storage resources and PVCs request them.

---

# Demo 1 — Demonstrating Data Loss with MySQL Deployment

## Objective

Deploy two MySQL Pods using a Deployment and `emptyDir`.

Create a database, table, and records. Delete the Pod and observe what happens to the data.

> This is a learning demonstration of Pod-scoped storage loss. Running two independent MySQL instances under one Deployment is not a production database architecture.

## Step 1 — Create `deployment-mysql.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mysql-deployment
spec:
  replicas: 2
  selector:
    matchLabels:
      app: mysql
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
        - name: mysql
          image: mysql:8.0
          env:
            - name: MYSQL_ROOT_PASSWORD
              value: "password"
            - name: MYSQL_DATABASE
              value: "testdb"
          ports:
            - containerPort: 3306
          volumeMounts:
            - name: mysql-storage
              mountPath: /var/lib/mysql
      volumes:
        - name: mysql-storage
          emptyDir: {}
```

Apply:

```bash
kubectl apply -f deployment-mysql.yaml
```

Check the Pods:

```bash
kubectl get pods -l app=mysql
```

Wait until the MySQL containers are ready before running database commands.

## Step 2 — Select a Pod

Pod names change between runs, so don't hardcode a Pod name from an earlier deployment.

Set a variable:

```bash
POD=$(kubectl get pods -l app=mysql \
  -o jsonpath='{.items[0].metadata.name}')

echo "$POD"
```

## Step 3 — Create a database

```bash
kubectl exec "$POD" -- \
  mysql -uroot -ppassword \
  -e "CREATE DATABASE testdb;"
```

The manifest already sets `MYSQL_DATABASE=testdb`, so the database may already exist. If it does, continue to the next step.

## Step 4 — Create a table

```bash
kubectl exec "$POD" -- \
  mysql -uroot -ppassword -D testdb \
  -e "CREATE TABLE test_table (id INT);"
```

## Step 5 — Insert data

```bash
kubectl exec "$POD" -- \
  mysql -uroot -ppassword -D testdb \
  -e "INSERT INTO test_table VALUES (1), (2), (3);"
```

Verify the records:

```bash
kubectl exec "$POD" -- \
  mysql -uroot -ppassword -D testdb \
  -e "SELECT * FROM test_table;"
```

Expected output:

```text
+------+
| id   |
+------+
|    1 |
|    2 |
|    3 |
+------+
```

## Step 6 — Delete the Pod

```bash
kubectl delete pod "$POD"
```

Watch the replacement Pods:

```bash
kubectl get pods -l app=mysql -w
```

Press `Ctrl+C` after the replacement is created.

## Step 7 — Check the data again

Select a replacement Pod:

```bash
NEW_POD=$(kubectl get pods -l app=mysql \
  -o jsonpath='{.items[0].metadata.name}')

echo "$NEW_POD"
```

Check for the table:

```bash
kubectl exec "$NEW_POD" -- \
  mysql -uroot -ppassword -D testdb \
  -e "SHOW TABLES;"
```

The table created in the deleted Pod's `emptyDir` will not exist in the replacement Pod's fresh storage.

You may see:

```text
ERROR 1146 (42S02):
Table 'testdb.test_table' doesn't exist
```

If the command selects the other original Pod, it may still show the table. This is because each MySQL Pod has its own independent `emptyDir` and its own database instance.

### What did we learn?

```text
Pod A
  └── emptyDir A
       └── Database and records

Delete Pod A
      |
      v
emptyDir A is deleted

Replacement Pod A
  └── New emptyDir
       └── Fresh database initialization
```

`emptyDir` survives a container restart inside the same Pod, but its data is removed when the Pod itself is removed.

**Conclusion:** Deployments are appropriate for interchangeable replicas, but databases need carefully designed persistent storage and, where required, database replication.

---

# Demo 2 — Python Flask Application with StatefulSet and Amazon EFS

## Objective

Deploy a Python Flask application with three StatefulSet replicas.

The application will:

- Display a form to collect Name, Age, and Gender.
- Save submitted records in `/data/data.json`.
- Store the JSON file on Amazon EFS.
- Share the same data among all three Pods.
- Expose the application through a LoadBalancer Service.
- Demonstrate stable Pod names and data persistence after Pod deletion.

### Architecture

```text
                       LoadBalancer
                            |
                            v
                       python-lb
                            |
              +-------------+-------------+
              |             |             |
              v             v             v
         python-stateful-0  -1            -2
              |             |             |
              +-------------+-------------+
                            |
                            v
                     Shared EFS PVC
                            |
                            v
                       Amazon EFS
                            |
                            v
                       data.json
```

All three Pods share the same EFS filesystem.

**Important:** This demo demonstrates StatefulSet identity and shared EFS storage together. Since every replica uses the same PVC, it does not demonstrate separate per-Pod PVCs created using `volumeClaimTemplates`.

## Step 1 — Prepare the EFS infrastructure

Reuse the EFS setup from your previous lesson if you already created it:

1. Create an EFS filesystem.
2. Create a Security Group allowing TCP `2049` from the EKS worker-node Security Group.
3. Create mount targets in the required Availability Zones.
4. Install the EFS CSI driver.
5. Verify that the CSI driver Pods are running.

Check:

```bash
kubectl get pods -n kube-system | grep efs-csi
```

Do not recreate these AWS resources if you already have a working setup.

## Step 2 — Create the Flask application

Create `app.py`.

Your application should:

1. Read and write the JSON file at `/data/data.json`.
2. Display an HTML form for Name, Age, and Gender.
3. Read existing entries and display them.
4. Append a new entry when the form is submitted.
5. Run on `0.0.0.0:5000`.

Use the Flask application code from your lab notes, including the dashboard HTML and styling. Ensure the file has valid Python indentation before building the image.

The storage path is important:

```python
DATA_FILE = "/data/data.json"
```

The `/data` directory will be mounted from EFS.

## Step 3 — Create a Dockerfile

Create `Dockerfile`:

```dockerfile
FROM python:3.9-slim

WORKDIR /app

COPY app.py /app/app.py

RUN pip install --no-cache-dir flask

EXPOSE 5000

CMD ["python", "app.py"]
```

## Step 4 — Build and push the Docker image

Log in to Docker Hub:

```bash
docker login
```

Build the image:

```bash
docker build -t kastrov/python-stateful-app:latest .
```

Push it:

```bash
docker push kastrov/python-stateful-app:latest
```

Use your own Docker Hub username if it differs from `kastrov`.

## Step 5 — Create the StorageClass, PV, and PVC

Create `efs-sc-pv-pvc.yaml`.

Replace `fs-xxxxxxxx` with your actual EFS filesystem ID.

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: efs-sc
provisioner: efs.csi.aws.com
volumeBindingMode: Immediate
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: efs-pv
spec:
  capacity:
    storage: 5Gi
  volumeMode: Filesystem
  accessModes:
    - ReadWriteMany
  persistentVolumeReclaimPolicy: Retain
  storageClassName: efs-sc
  csi:
    driver: efs.csi.aws.com
    volumeHandle: fs-xxxxxxxx
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: efs-claim
spec:
  accessModes:
    - ReadWriteMany
  storageClassName: efs-sc
  resources:
    requests:
      storage: 5Gi
```

Apply:

```bash
kubectl apply -f efs-sc-pv-pvc.yaml
```

Verify:

```bash
kubectl get sc
kubectl get pv
kubectl get pvc
```

The PVC should reach `Bound`.

**Note:** This is a statically defined PV that references an existing EFS filesystem. The `5Gi` value is Kubernetes capacity metadata here, not an EFS storage quota.

## Step 6 — Create the StatefulSet and Services

Create `statefulset.yaml`.

This manifest contains both the required Headless Service and the external LoadBalancer Service.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: python-service
spec:
  clusterIP: None
  selector:
    app: python-stateful
  ports:
    - name: web
      port: 5000
      targetPort: 5000
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: python-stateful
spec:
  serviceName: python-service
  replicas: 3
  selector:
    matchLabels:
      app: python-stateful
  template:
    metadata:
      labels:
        app: python-stateful
    spec:
      containers:
        - name: python-app
          image: kastrov/python-stateful-app:latest
          ports:
            - containerPort: 5000
          volumeMounts:
            - name: shared-data
              mountPath: /data
      volumes:
        - name: shared-data
          persistentVolumeClaim:
            claimName: efs-claim
---
apiVersion: v1
kind: Service
metadata:
  name: python-lb
spec:
  type: LoadBalancer
  selector:
    app: python-stateful
  ports:
    - port: 80
      targetPort: 5000
```

### Why two Services?

**Headless Service — `python-service`**

```yaml
clusterIP: None
```

It supports stable DNS identities for StatefulSet Pods.

**LoadBalancer Service — `python-lb`**

It exposes the application externally and distributes requests among the ready Pods.

Apply:

```bash
kubectl apply -f statefulset.yaml
```

## Step 7 — Verify the Deployment

Check the StatefulSet:

```bash
kubectl get statefulset
```

Check the Pods:

```bash
kubectl get pods -o wide
```

Expected Pod names:

```text
python-stateful-0
python-stateful-1
python-stateful-2
```

Check storage:

```bash
kubectl get pvc
kubectl get pv
kubectl get sc
```

Check Services:

```bash
kubectl get svc
```

Wait for the LoadBalancer address:

```bash
kubectl get svc python-lb -w
```

Once the external hostname is assigned, open it in your browser using HTTP.

## Step 8 — Test the application

Open the LoadBalancer URL.

Enter a sample record:

```text
Name: Rahul
Age: 25
Gender: Male
```

Submit the form.

The application writes the record to:

```text
/data/data.json
```

Because `/data` is mounted from EFS, the file is stored in the shared filesystem.

## Step 9 — Verify the same data from another Pod

Check the JSON file in the first Pod:

```bash
kubectl exec python-stateful-0 -- cat /data/data.json
```

Check another Pod:

```bash
kubectl exec python-stateful-1 -- cat /data/data.json
```

Check the third Pod:

```bash
kubectl exec python-stateful-2 -- cat /data/data.json
```

The file should contain the same shared records in each Pod.

**Note:** The simple JSON application is suitable for a lab, not concurrent production writes. Two simultaneous form submissions can overwrite one another because the application reads and rewrites the same JSON file without locking or transactional storage.

## Step 10 — Delete a StatefulSet Pod

Delete one Pod:

```bash
kubectl delete pod python-stateful-0
```

Check the Pods:

```bash
kubectl get pods -w
```

Kubernetes recreates the Pod with the same name:

```text
python-stateful-0
```

Check the data again:

```bash
kubectl exec python-stateful-0 -- cat /data/data.json
```

The records remain because the file is stored on EFS, not in the deleted Pod's temporary filesystem.

### What did we learn?

```text
Delete python-stateful-0
          |
          v
Kubernetes recreates python-stateful-0
          |
          v
New Pod mounts the same EFS filesystem
          |
          v
Existing data.json remains available
```

Stable Pod identity comes from the StatefulSet. Data persistence comes from EFS.

These are two different features working together.

## Step 11 — Update the Application

After changing `app.py`, rebuild and push the image:

```bash
docker build -t kastrov/python-stateful-app:latest .
docker push kastrov/python-stateful-app:latest
```

For a lab, trigger a rollout by updating the StatefulSet Pod template:

```bash
kubectl set image statefulset/python-stateful \
  python-app=kastrov/python-stateful:latest
```

Check rollout status:

```bash
kubectl rollout status statefulset/python-stateful
```

For reliable image updates, use a new version tag instead of repeatedly reusing `latest`, and update the manifest to reference that tag.

---

## Step 12 — Cleanup

Delete the StatefulSet and Services:

```bash
kubectl delete -f statefulset.yaml
```

Delete the PVC and PV manifest resources:

```bash
kubectl delete -f efs-sc-pv-pvc.yaml
```

The PV uses `persistentVolumeReclaimPolicy: Retain`, so the underlying EFS filesystem is not automatically deleted when the claim is removed.

If the CSI driver was installed only for this lab and no other workloads need it:

```bash
helm uninstall aws-efs-csi-driver -n kube-system
```

If you no longer need the EFS filesystem, delete its mount targets and then the filesystem through AWS after confirming that no workloads depend on it.

---

# Final Comparison — Deployment vs StatefulSet

| Feature | Deployment | StatefulSet |
|---|---|---|
| Pod names | Generated names | Predictable ordinal names |
| Pod identity | Interchangeable replicas | Stable identity |
| Storage | Configured separately | Can use PVCs and `volumeClaimTemplates` |
| Headless Service | Not required for ordinary Deployment use | Commonly used for stable Pod DNS |
| Typical workloads | Stateless web apps and APIs | Databases and stateful distributed apps |
| Automatic data replication | No | No |

## Interview Questions

### 1. Why use StatefulSet instead of Deployment?

A StatefulSet provides stable Pod identities, predictable names, ordered deployment behavior, and integration with persistent storage. These features are useful for applications such as databases that need stable identity.

### 2. Does StatefulSet automatically preserve data?

No. StatefulSet manages workload identity and lifecycle. Persistent storage, such as EBS or EFS, is required to preserve data beyond the lifetime of a Pod's temporary filesystem.

### 3. Why do we use a Headless Service?

A Headless Service supports stable DNS records for individual StatefulSet Pods, allowing applications to address particular Pods rather than only a load-balanced Service address.

### 4. What is `volumeClaimTemplates`?

It is a StatefulSet feature that creates a separate PVC for each Pod. This is useful when each database replica needs its own persistent volume.

### 5. What is the difference between the two demos?

- **Demo 1:** uses a Deployment and `emptyDir` to demonstrate why Pod-scoped storage does not preserve data after Pod deletion.
- **Demo 2:** uses a StatefulSet for stable Pod identities and EFS for shared persistent files across replicas.

## Easy Way to Remember

```text
Deployment
    ↓
Interchangeable Pods

StatefulSet
    ↓
Stable Pod identity

Headless Service
    ↓
Stable Pod DNS

PVC / PV
    ↓
Kubernetes storage resources

EBS / EFS
    ↓
Actual persistent storage
```
