# Kubernetes EFS Shared Storage with Nginx

## Overview

In this practical, we will deploy an Nginx application with three replicas in Amazon EKS and use Amazon EFS as shared persistent storage.

### Goal

- Multiple Pods share the same filesystem simultaneously.
- Use the `ReadWriteMany (RWX)` access mode.
- Store shared files that remain available when a Pod is recreated.
- Expose the Nginx application using a `LoadBalancer` Service.

### Architecture

```text
                 Internet
                    |
                    v
             LoadBalancer
                    |
                    v
              Nginx Service
                    |
          +---------+---------+
          |         |         |
          v         v         v
       Nginx-1   Nginx-2   Nginx-3
          |         |         |
          +---------+---------+
                    |
                    v
              EFS PVC
                    |
                    v
              Amazon EFS
```

All three Pods mount the same EFS filesystem at `/usr/share/nginx/html`.

---

## Prerequisites

- A running Amazon EKS cluster.
- AWS CLI configured with the required permissions.
- `kubectl` connected to the correct cluster.
- Helm installed.
- An EFS filesystem and mount targets in the VPC used by the EKS worker nodes.
- Network access from worker nodes to EFS on TCP port `2049`.

---

## Step 1 — Get the VPC ID

We need the VPC ID to create a Security Group for EFS.

```bash
VPC_ID=$(aws ec2 describe-subnets \
  --subnet-ids subnet-0899671d86d609265 \
  --query "Subnets[0].VpcId" \
  --output text)
```

Check the VPC ID:

```bash
echo $VPC_ID
```

**Why can we use one subnet ID?**

If both worker-node subnets belong to the same VPC, querying either subnet gives us the same VPC ID.

Replace the example subnet ID with a valid subnet from your own cluster.

---

## Step 2 — Create a Security Group for EFS

```bash
aws ec2 create-security-group \
  --group-name efs-sg \
  --description "Security group for EFS" \
  --vpc-id $VPC_ID
```

Save the returned `GroupId`, for example:

```text
sg-016a73a6ff7aaa30a
```

Set it as a shell variable:

```bash
EFS_SG_ID=<your-efs-security-group-id>
```

### Allow NFS traffic

EFS uses NFS over TCP port `2049`.

For a lab, allow inbound traffic from the EKS worker-node Security Group:

```bash
aws ec2 authorize-security-group-ingress \
  --group-id $EFS_SG_ID \
  --protocol tcp \
  --port 2049 \
  --source-group <Worker-Node-Security-Group-ID>
```

Replace the placeholder with the worker-node Security Group ID.

Avoid using `0.0.0.0/0` for NFS access. Restrict access to the required clients.

---

## Step 3 — Create an EFS File System

```bash
aws efs create-file-system \
  --creation-token kastro-efs \
  --performance-mode generalPurpose \
  --throughput-mode bursting
```

The output includes a File System ID, for example:

```text
fs-03e917586b79c8118
```

Save it:

```bash
EFS_FS_ID=<your-efs-file-system-id>
```

Replace the placeholder with the actual ID returned by AWS.

Wait until the filesystem is available before creating mount targets.

---

## Step 4 — Create EFS Mount Targets

Mount targets allow clients in the VPC to connect to EFS.

Create a mount target in the first worker-node subnet:

```bash
aws efs create-mount-target \
  --file-system-id $EFS_FS_ID \
  --subnet-id <Node1-SubnetID> \
  --security-groups $EFS_SG_ID
```

Create another in a subnet belonging to a different Availability Zone:

```bash
aws efs create-mount-target \
  --file-system-id $EFS_FS_ID \
  --subnet-id <Node2-SubnetID> \
  --security-groups $EFS_SG_ID
```

Each EFS filesystem can have one mount target per Availability Zone in a VPC. Ensure the subnets are in different AZs and that the security groups permit NFS traffic.

Verify:

```bash
aws efs describe-mount-targets \
  --file-system-id $EFS_FS_ID
```

---

## Step 5 — Install the EFS CSI Driver

The EFS CSI driver allows Kubernetes to mount Amazon EFS volumes into Pods.

### Install Helm if necessary

```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
```

Verify:

```bash
helm version
```

### Add the Helm repository

```bash
helm repo add aws-efs-csi-driver \
  https://kubernetes-sigs.github.io/aws-efs-csi-driver/
```

```bash
helm repo update
```

### Install the driver

```bash
helm install aws-efs-csi-driver \
  aws-efs-csi-driver/aws-efs-csi-driver \
  --namespace kube-system \
  --create-namespace
```

The EFS CSI driver also needs the required AWS permissions and network access. Configure its IAM permissions according to the EKS setup.

Verify:

```bash
kubectl get pods -n kube-system | grep efs-csi
```

The driver components should be running before continuing.

---

## Step 6 — Create the StorageClass

Create a file named `efs-sc.yaml`.

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: efs-sc
provisioner: efs.csi.aws.com
volumeBindingMode: Immediate
```

Apply it:

```bash
kubectl apply -f efs-sc.yaml
```

Verify:

```bash
kubectl get storageclass
```

**Important:** This StorageClass identifies the EFS CSI provisioner. The static PV used in this practical refers directly to an existing EFS filesystem; this is not dynamic EFS filesystem provisioning.

---

## Step 7 — Create the PV and PVC

Create a file named `efs-pv-pvc.yaml`.

Replace `fs-12345678` with your actual EFS filesystem ID.

```yaml
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
    volumeHandle: fs-12345678
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
kubectl apply -f efs-pv-pvc.yaml
```

Verify:

```bash
kubectl get pv
kubectl get pvc
```

Expected result:

```text
NAME     CAPACITY   ACCESS MODES   STATUS   CLAIM
efs-pv   5Gi        RWX            Bound    default/efs-claim
```

The PVC should show `Bound`.

**Important:** The `5Gi` capacity here is Kubernetes PV/PVC metadata for this static EFS example. It does not set a 5Gi quota on the actual EFS filesystem.

---

## Step 8 — Deploy Nginx with Three Replicas

Create `nginx-deployment.yaml`.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-shared
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx-shared
  template:
    metadata:
      labels:
        app: nginx-shared
    spec:
      containers:
        - name: nginx
          image: nginx:stable
          ports:
            - containerPort: 80
          volumeMounts:
            - name: shared-storage
              mountPath: /usr/share/nginx/html
      volumes:
        - name: shared-storage
          persistentVolumeClaim:
            claimName: efs-claim
---
apiVersion: v1
kind: Service
metadata:
  name: nginx-lb
spec:
  type: LoadBalancer
  selector:
    app: nginx-shared
  ports:
    - port: 80
      targetPort: 80
```

Apply:

```bash
kubectl apply -f nginx-deployment.yaml
```

Check the Pods:

```bash
kubectl get pods -o wide
```

Check the Deployment:

```bash
kubectl get deployment nginx-shared
```

Check the Service:

```bash
kubectl get svc nginx-lb
```

The Pods can run on different worker nodes and share the same EFS filesystem. Their exact placement depends on scheduling and available cluster capacity.

---

## Step 9 — Why Does Nginx Return 403 Forbidden?

Nginx normally serves files from:

```text
/usr/share/nginx/html
```

The PVC mount replaces the contents visible at that path with the contents of EFS.

Initially, the EFS directory may be empty:

```text
EFS
└── empty directory
```

There is no `index.html` file for Nginx to serve. When directory listing is not enabled, Nginx can return:

```text
403 Forbidden
```

This does not necessarily mean the LoadBalancer or Service is broken. It may simply mean the shared directory has no index file.

---

## Step 10 — Create a Shared index.html File

First, list the Pods:

```bash
kubectl get pods -l app=nginx-shared
```

Choose one Pod and open a shell:

```bash
kubectl exec -it <pod-name> -- sh
```

Create the file:

```sh
echo "Hello from Kastro's shared EFS!" > /usr/share/nginx/html/index.html
```

Exit the shell:

```sh
exit
```

Because the directory is backed by EFS, the file is stored in the shared filesystem rather than only in the container's writable layer.

---

## Step 11 — Test Shared Storage

### Test through the LoadBalancer

Get the Service address:

```bash
kubectl get svc nginx-lb
```

Wait until `EXTERNAL-IP` displays a hostname or address. For an AWS LoadBalancer, it may take a few minutes.

Open the address in a browser, or test from a terminal:

```bash
curl http://<LoadBalancer-DNS>
```

Expected response:

```text
Hello from Kastro's shared EFS!
```

### Verify from another Pod

Get another Pod name:

```bash
kubectl get pods -l app=nginx-shared
```

Read the file from a different Pod:

```bash
kubectl exec <another-pod-name> -- \
  cat /usr/share/nginx/html/index.html
```

Expected output:

```text
Hello from Kastro's shared EFS!
```

This confirms that both Pods can access the same file.

---

## Step 12 — Test Persistence After Pod Deletion

Delete one Nginx Pod:

```bash
kubectl delete pod <pod-name>
```

The Deployment creates a replacement Pod to maintain the desired replica count.

Check:

```bash
kubectl get pods -o wide
```

Read the file from the replacement Pod:

```bash
kubectl exec <replacement-pod-name> -- \
  cat /usr/share/nginx/html/index.html
```

Expected output:

```text
Hello from Kastro's shared EFS!
```

The file remains because it is stored on EFS, which exists independently of the deleted Pod.

---

## Step 13 — Cleanup

Delete the Nginx Deployment and Service:

```bash
kubectl delete -f nginx-deployment.yaml
```

Delete the PVC and PV:

```bash
kubectl delete -f efs-pv-pvc.yaml
```

Delete the StorageClass:

```bash
kubectl delete -f efs-sc.yaml
```

Uninstall the CSI driver only if no other workloads need it:

```bash
helm uninstall aws-efs-csi-driver -n kube-system
```

### AWS resource cleanup

The Kubernetes cleanup commands do not delete the EFS filesystem, mount targets, or Security Group.

If you no longer need the EFS filesystem:

1. Ensure no workloads are using it.
2. Delete its mount targets.
3. Wait for mount-target deletion to complete.
4. Delete the EFS filesystem.
5. Delete the EFS Security Group if it is no longer used.

Because the PV uses `persistentVolumeReclaimPolicy: Retain`, deleting the PVC does not automatically delete the underlying EFS filesystem.

---

## Key Learnings

- EFS provides shared persistent file storage.
- `ReadWriteMany` allows multiple nodes to mount a volume read-write.
- A PVC is a request for storage, and a PV represents the Kubernetes storage resource.
- The EFS CSI driver connects Kubernetes to Amazon EFS.
- Multiple Nginx Pods can read and write the same EFS-backed directory.
- Data stored on EFS survives Pod deletion and recreation.
- A LoadBalancer Service exposes the Nginx application externally.
- An empty mounted web directory can cause Nginx to return `403 Forbidden` because no index file exists.
- The example uses a **statically defined PV backed by an existing EFS filesystem**, even though the StorageClass is configured.

### Interview Question

**How can multiple Kubernetes Pods share persistent storage across different worker nodes?**

We can use Amazon EFS with the EFS CSI driver and a PersistentVolumeClaim configured for `ReadWriteMany`. Each Pod mounts the same EFS filesystem, allowing the Pods to access shared files across worker nodes and Availability Zones.
