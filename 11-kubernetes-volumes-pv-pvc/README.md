# Kubernetes Volumes, PV, PVC, StorageClass, EBS and EFS

Kubernetes Pods are temporary. If data is stored only inside a Pod's container filesystem, that data can be lost when the Pod is deleted and recreated.

For applications that need persistent data, Kubernetes provides **Volumes, PersistentVolumes (PV), PersistentVolumeClaims (PVC), and StorageClasses**.

---

# 1. Why Do We Need Persistent Storage?

Consider a database running inside a Pod:

```text
Pod
└── Database
    └── Data
```

If the Pod is deleted:

```text
Pod deleted
     ↓
Container filesystem deleted
     ↓
Data may be lost
```

Therefore, applications such as databases need persistent storage outside the temporary Pod/container filesystem.

---

# 2. Stateless vs Stateful Applications

### Stateless application

A stateless application does not depend on data stored inside the individual Pod.

Example:

```text
Frontend
API Gateway
Web Server
```

If a frontend Pod is deleted, Kubernetes can create another Pod without needing the old Pod's local data.

### Stateful application

A stateful application maintains persistent state or data.

Examples:

```text
MySQL
PostgreSQL
MongoDB
Redis
```

These applications commonly require persistent storage.

> Stateful does not simply mean "the data survives Pod deletion." Stateful applications maintain persistent state, identity, or ordering, and persistent storage is one important mechanism used to preserve their data.

---

# 3. Kubernetes Volumes

A Kubernetes Volume provides storage that can be mounted inside a Pod.

Example:

```yaml
spec:
  containers:
    - name: nginx
      image: nginx

      volumeMounts:
        - name: app-storage
          mountPath: /data

  volumes:
    - name: app-storage
      emptyDir: {}
```

Here:

```text
Pod
 │
 └── Volume
      │
      └── /data
```

The type of Volume determines how and where the data is stored.

---

# 4. PersistentVolume (PV)

A **PersistentVolume (PV)** is a piece of storage made available to the Kubernetes cluster.

It can be backed by storage such as:

```text
Amazon EBS
Amazon EFS
Azure Disk
Google Persistent Disk
NFS
etc.
```

A PV exists independently from a particular Pod.

```text
Cluster
   │
   └── PV
        │
        └── Persistent Storage
```

The storage administrator or a dynamic provisioning system can create the PV.

---

# 5. PersistentVolumeClaim (PVC)

A **PersistentVolumeClaim (PVC)** is a request for storage made by a Kubernetes user/application.

For example:

```text
Application
     │
     ▼
    PVC
     │
     ▼
   Request
  "I need 10Gi"
     │
     ▼
    PV
```

The PVC does **not** contain the actual storage.

It is a request for storage.

---

# 6. PV vs PVC

| PV | PVC |
|---|---|
| Actual storage resource | Request for storage |
| Cluster resource | User/application request |
| Provides storage | Requests storage |
| Can be created statically or dynamically | Created by application/user |
| Example: 20Gi EBS-backed PV | Request for 10Gi |

### Easy analogy

Think of a hotel:

```text
PV  → Hotel room
PVC → Customer booking/request
Pod → Customer staying in the room
```

The customer does not create the hotel room.

The customer requests a room, and Kubernetes matches the request with available storage.

---

# 7. Access Modes

PVCs specify how the storage should be accessed.

### RWO — ReadWriteOnce

The volume can be mounted as read-write by workloads on **one node at a time**.

Common example:

```text
EBS
```

### ROX — ReadOnlyMany

The volume can be mounted as read-only by multiple nodes.

### RWX — ReadWriteMany

The volume can be mounted as read-write by multiple nodes.

Common example:

```text
EFS
```

### Easy memory trick

```text
RWO → ReadWriteOnce
ROX → ReadOnlyMany
RWX → ReadWriteMany
```

---

# 8. Static vs Dynamic Provisioning

There are two major ways to create persistent storage.

## Static Provisioning

The administrator creates the storage first.

```text
Administrator
     │
     ▼
    PV
     │
     ▼
    PVC
     │
     ▼
    Pod
```

The PV already exists before the application requests it.

### Problems with static provisioning

- Manual work
- Difficult to manage at scale
- Possible resource wastage
- Administrator must create storage manually

---

# 9. Dynamic Provisioning

With dynamic provisioning, Kubernetes can create storage automatically when a PVC requests it.

```text
Pod
 │
 ▼
PVC
 │
 ▼
StorageClass
 │
 ▼
CSI Driver
 │
 ▼
Cloud Storage
```

This is the preferred approach in many cloud environments.

---

# 10. StorageClass

A **StorageClass** defines how storage should be dynamically provisioned.

It specifies things such as:

- Storage provisioner/CSI driver
- Storage parameters
- Reclaim policy
- Other backend-specific options

Example:

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: my-storage
provisioner: ebs.csi.aws.com
```

Then a PVC can request that StorageClass:

```yaml
spec:
  storageClassName: my-storage
```

---

# 11. CSI — Container Storage Interface

CSI stands for:

**Container Storage Interface**

CSI allows Kubernetes to communicate with external storage systems.

For AWS:

```text
Kubernetes
     │
     ▼
CSI Driver
     │
     ├── EBS CSI
     │
     └── EFS CSI
```

The CSI driver communicates with the AWS storage service.

---

# 12. Amazon EBS

Amazon EBS is **block storage**.

It is commonly used for:

- Databases
- Application disks
- Persistent block storage
- Workloads requiring high-performance block devices

Architecture:

```text
Pod
 │
 ▼
PVC
 │
 ▼
StorageClass
 │
 ▼
EBS CSI Driver
 │
 ▼
Amazon EBS Volume
```

---

# 13. Scenario A — Dynamic Provisioning with EBS CSI

The EBS CSI driver allows Kubernetes to dynamically provision Amazon EBS volumes.

## Prerequisites

The EBS CSI driver needs AWS permissions to perform operations such as:

```text
Create volume
Attach volume
Detach volume
Delete volume
```

For modern EKS setups, the recommended approach is to give the **EBS CSI controller** a dedicated IAM role using **IRSA or EKS Pod Identity**.

A simple lab may use the worker-node IAM role, but a dedicated CSI-driver IAM role is the better production pattern.

---

# 14. Associate IAM OIDC Provider

The OIDC provider allows a Kubernetes ServiceAccount to assume an AWS IAM role.

```bash
eksctl utils associate-iam-oidc-provider \
  --cluster kastro-cluster \
  --region us-east-1 \
  --approve
```

This establishes a relationship like:

```text
Kubernetes ServiceAccount
          │
          ▼
     OIDC Provider
          │
          ▼
       IAM Role
          │
          ▼
      AWS APIs
```

This avoids putting AWS access keys directly inside Pods.

---

# 15. Check EBS CSI Driver

Check whether the EBS CSI driver is already installed:

```bash
kubectl get pods -n kube-system | grep ebs-csi
```

You should see components such as:

```text
ebs-csi-controller-xxxxx
ebs-csi-node-xxxxx
```

They should be in `Running` state.

---

# 16. Install Helm

If Helm is not already installed:

```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
```

Check:

```bash
helm version
```

---

# 17. Add EBS CSI Helm Repository

```bash
helm repo add aws-ebs-csi-driver \
  https://kubernetes-sigs.github.io/aws-ebs-csi-driver
```

Update the repository:

```bash
helm repo update
```

This downloads the latest chart information from the configured repository.

---

# 18. Install EBS CSI Driver

```bash
helm install aws-ebs-csi-driver \
  aws-ebs-csi-driver/aws-ebs-csi-driver \
  --namespace kube-system \
  --create-namespace
```

The Helm chart provides the required EBS CSI components.

---

# 19. Verify EBS CSI Driver

```bash
kubectl get pods -n kube-system | grep ebs-csi
```

You can also check the DaemonSet:

```bash
kubectl get daemonset -n kube-system | grep ebs-csi
```

And the controller Deployment:

```bash
kubectl get deployment -n kube-system | grep ebs-csi
```

---

# 20. Find Which Pod Is Using Which PVC

Useful command:

```bash
kubectl get pod \
  -o custom-columns=POD:.metadata.name,PVC:.spec.volumes[*].persistentVolumeClaim.claimName
```

Example:

```text
POD                  PVC
movievault-pod       movievault-pvc
nginx-pod            nginx-pvc
```

To hide Pods that don't use PVCs:

```bash
kubectl get pod \
  -o custom-columns=POD:.metadata.name,PVC:.spec.volumes[*].persistentVolumeClaim.claimName \
  | grep -v "<none>"
```

---

# 21. Important EBS Limitation

EBS is **Availability-Zone scoped block storage**.

For example:

```text
us-east-1a
   │
   └── EBS Volume
        │
        └── Node / Pod
```

The EBS volume is associated with a particular Availability Zone.

Therefore, a workload using EBS must be scheduled appropriately so that the Pod can access the volume.

EBS is not designed as a general shared filesystem across multiple AZs.

For shared storage across multiple nodes/AZs, EFS is usually more suitable.

---

# 22. Amazon EFS

Amazon EFS stands for:

**Elastic File System**

EFS is a managed **NFS-based shared file system**.

It is useful when multiple Pods need to access the same files.

Example:

```text
Worker Node 1
    │
   Pod A
    │
    ├──────────┐
               │
Worker Node 2  │
    │          │
   Pod B       ├──► EFS
               │
Worker Node 3  │
    │          │
   Pod C       │
               │
               └──────────
```

Multiple Pods can access the same EFS file system.

---

# 23. Why Use EFS?

Suppose:

```text
Pod A → Node 1 → AZ-a

Pod B → Node 2 → AZ-b
```

If both Pods need to access the same files, EFS can provide shared storage.

Example use cases:

- Shared application files
- User uploads
- Shared configuration/data
- Content repositories
- Applications requiring RWX storage

---

# 24. Scenario B — Dynamic Provisioning with EFS CSI

The EFS CSI driver allows Kubernetes to communicate with Amazon EFS.

Architecture:

```text
Pod A ─────┐
Pod B ─────┼──► PVC
Pod C ─────┘
             │
             ▼
        StorageClass
             │
             ▼
        EFS CSI Driver
             │
             ▼
            EFS
```

---

# 25. Step 1 — Get VPC ID

First identify a subnet used by an EKS worker node.

Then:

```bash
VPC_ID=$(aws ec2 describe-subnets \
  --subnet-ids <Node1-SubnetID> \
  --query "Subnets[0].VpcId" \
  --output text)
```

Check:

```bash
echo $VPC_ID
```

If the worker-node subnets belong to the same VPC, using one worker-node subnet is enough to retrieve the VPC ID.

---

# 26. Step 2 — Create Security Group

Create a Security Group for EFS:

```bash
aws ec2 create-security-group \
  --group-name efs-sg \
  --description "Security group for EFS" \
  --vpc-id $VPC_ID
```

The output will contain a Security Group ID, for example:

```text
sg-01ab1f1033a19b4c1
```

Save this ID.

---

# 27. Allow NFS Traffic

EFS uses:

```text
TCP port 2049
```

For a lab environment:

```bash
aws ec2 authorize-security-group-ingress \
  --group-id <GroupID-From-Above-Step> \
  --protocol tcp \
  --port 2049 \
  --cidr 0.0.0.0/0
```

### Production recommendation

Do not expose NFS to the entire internet.

Instead, allow port `2049` only from the EKS worker-node Security Group.

```text
EKS Worker Nodes
       │
       │ TCP 2049
       ▼
    EFS SG
       │
       ▼
      EFS
```

---

# 28. Step 3 — Create EFS File System

Create the EFS file system:

```bash
aws efs create-file-system \
  --creation-token kastro-efs \
  --performance-mode generalPurpose \
  --throughput-mode bursting
```

The response contains a File System ID:

```text
fs-03e917586b79c8118
```

Save this ID.

---

# 29. Step 4 — Create EFS Mount Targets

An EFS file system needs mount targets in the Availability Zones where clients need access.

For example:

```text
AZ-a
 │
 └── Subnet A
      │
      └── EFS Mount Target


AZ-b
 │
 └── Subnet B
      │
      └── EFS Mount Target
```

Create the first mount target:

```bash
aws efs create-mount-target \
  --file-system-id <FileSystemID> \
  --subnet-id <Subnet-ID-of-Node1> \
  --security-groups <Security-Group-ID>
```

Create another mount target in a different AZ/subnet:

```bash
aws efs create-mount-target \
  --file-system-id <FileSystemID> \
  --subnet-id <Subnet-ID-of-Node2> \
  --security-groups <Security-Group-ID>
```

> Use subnets from the Availability Zones where your EKS worker nodes run.

---

# 30. Step 5 — Verify Mount Targets

```bash
aws efs describe-mount-targets \
  --file-system-id <FileSystemID>
```

You should see the configured mount targets along with their:

- Mount Target ID
- Subnet
- Availability Zone
- Security Group
- IP address

---

# 31. Step 6 — Install EFS CSI Driver

Add the Helm repository:

```bash
helm repo add aws-efs-csi-driver \
  https://kubernetes-sigs.github.io/aws-efs-csi-driver/
```

Update:

```bash
helm repo update
```

Install:

```bash
helm install aws-efs-csi-driver \
  aws-efs-csi-driver/aws-efs-csi-driver \
  --namespace kube-system \
  --create-namespace
```

---

# 32. Verify EFS CSI Driver

```bash
kubectl get pods -n kube-system | grep efs-csi
```

You should see the EFS CSI components running.

Check the DaemonSet:

```bash
kubectl get daemonset -n kube-system | grep efs-csi
```

---

# 33. EBS vs EFS

| Feature | EBS | EFS |
|---|---|---|
| Full name | Elastic Block Store | Elastic File System |
| Storage type | Block storage | File storage |
| Shared across nodes | Limited | Yes |
| Multi-AZ | Volume is AZ-scoped | Designed for multi-AZ access |
| Common access mode | RWO | RWX |
| Typical use | Databases, application disks | Shared files |
| CSI driver | EBS CSI | EFS CSI |
| Example | PostgreSQL data | Shared uploads |

---

# 34. Complete EBS Storage Flow

```text
                    Kubernetes
                        │
                        ▼
                       Pod
                        │
                        ▼
                       PVC
                        │
                        ▼
                  StorageClass
                        │
                        ▼
                  EBS CSI Driver
                        │
                        ▼
                   Amazon EBS
```

---

# 35. Complete EFS Storage Flow

```text
                Kubernetes
                     │
          ┌──────────┼──────────┐
          ▼          ▼          ▼
        Pod A      Pod B      Pod C
          │          │          │
          └──────────┼──────────┘
                     ▼
                    PVC
                     │
                     ▼
               StorageClass
                     │
                     ▼
               EFS CSI Driver
                     │
                     ▼
                    EFS
```

---

# 36. Static vs Dynamic Provisioning

```text
STATIC PROVISIONING

Administrator
     │
     ▼
    PV
     │
     ▼
    PVC
     │
     ▼
    Pod
```

```text
DYNAMIC PROVISIONING

Pod
 │
 ▼
PVC
 │
 ▼
StorageClass
 │
 ▼
CSI Driver
 │
 ▼
Cloud Storage
 │
 ▼
PV
```

The major advantage of dynamic provisioning is that the storage can be created automatically when required.

---

# 37. Important Commands

### Check StorageClasses

```bash
kubectl get storageclass
```

### Check PVs

```bash
kubectl get pv
```

### Check PVCs

```bash
kubectl get pvc
```

### Check PVCs in all namespaces

```bash
kubectl get pvc -A
```

### Describe a PVC

```bash
kubectl describe pvc <PVC_NAME>
```

### Describe a PV

```bash
kubectl describe pv <PV_NAME>
```

### Check Pods using PVCs

```bash
kubectl get pod \
  -o custom-columns=POD:.metadata.name,PVC:.spec.volumes[*].persistentVolumeClaim.claimName
```

---

# 38. Easy Way to Remember

```text
PV
 ↓
Actual persistent storage


PVC
 ↓
Request for storage


StorageClass
 ↓
Defines how storage should be dynamically created


CSI Driver
 ↓
Connects Kubernetes to the storage provider


EBS
 ↓
Block storage
 ↓
Usually RWO
 ↓
Good for databases/application disks


EFS
 ↓
Shared file storage
 ↓
RWX
 ↓
Good for shared files across Pods/nodes
```

---

# 39. Interview Question

### Why do we need PVC if PV already provides storage?

A **PV represents the actual storage resource**, while a **PVC is a request for that storage**.

The application normally uses the PVC instead of directly referring to a specific PV.

```text
Application
     │
     ▼
    PVC
     │
     ▼
    PV
     │
     ▼
Storage Backend
```

This separates the application from the underlying storage implementation.

---

### Why do we use StorageClass?

A StorageClass defines how Kubernetes should dynamically provision storage.

Instead of an administrator manually creating a PV every time an application needs storage:

```text
PVC
 ↓
StorageClass
 ↓
CSI Driver
 ↓
Storage automatically created
```

---

### Why EFS instead of EBS?

If multiple Pods running on different nodes/AZs need to read and write the same files, EFS is a better choice because it provides shared file storage.

EBS is block storage and is AZ-scoped, making it more suitable for workloads that need persistent block storage rather than a shared filesystem.

---

### What is CSI?

CSI stands for **Container Storage Interface**.

It provides a standard interface that allows Kubernetes to communicate with external storage systems.

For AWS:

```text
Kubernetes
    │
    ├── EBS CSI Driver ──► EBS
    │
    └── EFS CSI Driver ──► EFS
```

---

# Final Memory Trick

```text
PVC = "I need storage"

StorageClass = "How should I get it?"

CSI = "Who connects Kubernetes to the storage?"

PV = "Here is the storage resource"

EBS = "Block storage"

EFS = "Shared file storage"
```
