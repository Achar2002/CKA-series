# Kubernetes Volumes, PV, PVC and Storage Classes

When data is written inside a Pod's container filesystem, that data is tied to the lifecycle of the Pod/container.

If the Pod is deleted and a new Pod is created, data stored only inside the old Pod's writable container filesystem may be lost.

This creates the need for **persistent storage**.

Kubernetes provides storage concepts such as:

- Volumes
- PersistentVolume (PV)
- PersistentVolumeClaim (PVC)
- StorageClass
- CSI Drivers

---

# 1. Why Do We Need Persistent Storage?

Consider a Pod running a database:

```text
Database Pod
     |
     ↓
Writes data
     |
     ↓
Pod deleted
     |
     ↓
New Pod created
     |
     ↓
Data stored only inside old container filesystem
     |
     ↓
Data may be lost
```

Therefore, applications that need data to survive Pod replacement should use persistent storage.

---

# 2. Stateless vs Stateful Applications

## Stateless Application

A stateless application does not depend on locally stored state that must survive replacement of the Pod.

For example:

```text
Zomato Frontend
      ↓
Stateless
```

If the frontend Pod is deleted, another Pod can be created and serve the application without needing the old Pod's local filesystem data.

---

## Stateful Application

A stateful application maintains important data or identity that must persist across Pod replacement.

For example:

```text
Zomato Database
      ↓
Stateful
```

The database data should be stored on persistent storage rather than only inside the Pod's writable filesystem.

---

# 3. Persistent Storage

Kubernetes can connect workloads to storage outside the Pod.

A simplified view:

```text
Pod
 |
 | mounts
 ↓
PVC
 |
 | binds to
 ↓
PV
 |
 | backed by
 ↓
Storage
```

For example, in AWS:

```text
Pod
 ↓
PVC
 ↓
PV
 ↓
EBS Volume
```

Or for shared file storage:

```text
Pod
 ↓
PVC
 ↓
PV
 ↓
EFS
```

---

# 4. PV and PVC

There are two important building blocks:

### PV — PersistentVolume

A **PersistentVolume** represents storage available to the Kubernetes cluster.

### PVC — PersistentVolumeClaim

A **PersistentVolumeClaim** is a request from a user/application for storage.

Think of it like a hotel:

```text
PV  → Hotel room
PVC → Customer booking/request
Pod → Customer using the room
```

The user does not normally need to manage the underlying storage directly.

---

# 5. PV and PVC Relationship

The basic flow is:

```text
Storage
   ↓
PersistentVolume
   ↓
PersistentVolumeClaim
   ↓
Pod
```

The Pod uses the PVC.

The PVC binds to a PV.

The PV represents the actual storage resource.

---

# 6. PVs Are Independent of Pods

A PV is not directly tied to a Pod.

A PV can exist even when no Pod is currently using it.

For example:

```text
PV
 |
 └── PVC
      |
      └── Pod
```

If the Pod is deleted, the PV does not automatically disappear just because the Pod is gone.

The behavior of the PV after PVC deletion depends on its **reclaim policy**.

---

# 7. Provisioning Storage

There are two main ways to create PVs:

```text
1. Static Provisioning
2. Dynamic Provisioning
```

---

# 8. Static Provisioning

In **static provisioning**, the administrator manually creates PVs.

Example:

```text
Administrator
      |
      ↓
Creates PV
      |
      ↓
User creates PVC
      |
      ↓
Kubernetes searches for matching PV
      |
      ↓
PVC binds to PV
```

If no suitable PV exists:

```text
PVC
 ↓
Pending
```

---

# 9. Why Static Provisioning Is Less Convenient

Static provisioning requires manual work.

For example:

```text
Administrator
    ↓
Create storage
    ↓
Create PV
    ↓
Wait for PVC
```

This becomes difficult when many applications need storage.

Possible problems include:

- Manual effort
- Difficult to manage at scale
- Storage may be underutilized
- Administrator must provision PVs ahead of time

---

# 10. Dynamic Provisioning

In **dynamic provisioning**, Kubernetes automatically creates storage when a PVC requests it.

The basic flow is:

```text
User creates PVC
       ↓
PVC references StorageClass
       ↓
StorageClass uses CSI driver/provisioner
       ↓
Backend storage is created
       ↓
PV is created
       ↓
PVC binds to PV
       ↓
Pod uses PVC
```

The administrator does not need to manually create every PV.

---

# 11. StorageClass

A **StorageClass** defines how storage should be dynamically provisioned.

It acts like a template for creating storage.

A StorageClass can define things such as:

- Provisioner/CSI driver
- Storage type
- Backend parameters
- Reclaim policy
- Volume binding behavior
- Other storage-specific options

The simplified relationship is:

```text
PVC
 ↓
StorageClass
 ↓
CSI Driver
 ↓
Storage Backend
 ↓
PV
```

---

# 12. CSI Driver

CSI stands for:

**Container Storage Interface**

CSI drivers allow Kubernetes to communicate with different storage systems.

For example:

```text
Kubernetes
     |
     ↓
CSI Driver
     |
     ├── AWS EBS
     ├── AWS EFS
     ├── Azure Disk
     ├── Azure Files
     └── Other storage systems
```

For AWS:

```text
EBS CSI Driver → EBS volumes
EFS CSI Driver → EFS file systems
```

---

# 13. Access Modes

Access modes define how a volume can be mounted by workloads.

The three commonly discussed access modes are:

```text
RWO
ROX
RWX
```

---

## RWO — ReadWriteOnce

```text
Read + Write
```

The volume can be mounted read-write by workloads on **one node at a time**.

Example:

```text
Node 1
 ├── Pod A
 └── Pod B
       |
       ↓
      EBS
```

Multiple Pods on the same node may be able to use the volume, depending on the workload and mount configuration.

But the volume cannot generally be mounted read-write from multiple nodes simultaneously.

AWS EBS is commonly used with:

```text
ReadWriteOnce
```

---

## ROX — ReadOnlyMany

```text
Read Only + Multiple
```

Multiple nodes can mount the volume as read-only.

```text
Node 1 ──┐
Node 2 ──┼──→ Storage
Node 3 ──┘
       Read Only
```

---

## RWX — ReadWriteMany

```text
Read + Write + Multiple
```

Multiple nodes can mount the same volume for read/write access.

```text
Node 1 ──┐
Node 2 ──┼──→ Shared Storage
Node 3 ──┘
       Read + Write
```

Shared file systems such as AWS EFS are commonly used for this type of workload.

> Important: The access modes a storage system supports depend on the storage backend and CSI driver. Not every storage system supports every access mode.

---

# 14. Access Mode Comparison

| Access Mode | Read | Write | Multiple Nodes |
|---|---|---|---|
| RWO | Yes | Yes | No, one node at a time |
| ROX | Yes | No | Yes |
| RWX | Yes | Yes | Yes |

Easy memory trick:

```text
RWO → Read Write Once
ROX → Read Only Many
RWX → Read Write Many
```

---

# 15. Reclaim Policy

A reclaim policy defines what happens to the PV after its PVC is deleted.

It controls the lifecycle behavior of the underlying storage/PV.

Common policies are:

```text
1. Retain
2. Delete
3. Recycle (deprecated)
```

---

# 16. Retain

```yaml
persistentVolumeReclaimPolicy: Retain
```

The PV and underlying storage are retained after the PVC is deleted.

This is useful when we want to protect important data.

```text
PVC deleted
     ↓
PV retained
     ↓
Storage/data preserved
```

The administrator can then manually handle the PV and storage.

---

# 17. Delete

```yaml
persistentVolumeReclaimPolicy: Delete
```

When the PVC is deleted, Kubernetes can delete the dynamically provisioned PV and associated backend storage according to the driver's behavior.

```text
PVC deleted
     ↓
PV deleted
     ↓
Backend storage may be deleted
```

This is useful when storage should have the same lifecycle as the claim.

---

# 18. Recycle

```text
Recycle
```

The old `Recycle` policy performed a basic cleanup of the volume and made it available for reuse.

However:

> **Recycle is deprecated and should not be used for modern Kubernetes configurations.**

The commonly used policies today are:

```text
Retain
Delete
```

---

# 19. Static vs Dynamic Provisioning

| Feature | Static | Dynamic |
|---|---|---|
| PV creation | Administrator | Automatically provisioned |
| StorageClass | Not required | Usually used |
| Manual effort | High | Low |
| Suitable for | Pre-created storage | On-demand storage |
| Scalability | Lower | Higher |

### Static

```text
Admin
 ↓
PV
 ↓
PVC
 ↓
Pod
```

### Dynamic

```text
PVC
 ↓
StorageClass
 ↓
CSI Driver
 ↓
Storage Backend
 ↓
PV
 ↓
Pod
```

---

# 20. Scenario A — Dynamic Provisioning with AWS EBS

AWS EBS is block storage.

A common EKS storage architecture is:

```text
Pod
 ↓
PVC
 ↓
StorageClass
 ↓
EBS CSI Driver
 ↓
AWS EBS Volume
```

The EBS CSI driver allows Kubernetes to manage EBS volumes.

---

# 21. EBS CSI Driver Prerequisites

Before using EBS dynamic provisioning, make sure the cluster has the AWS EBS CSI driver configured.

The driver needs appropriate AWS permissions to perform operations such as:

- Create volumes
- Attach volumes
- Detach volumes
- Delete volumes

The exact IAM configuration depends on how the EBS CSI driver is installed.

For example, when using IAM roles for service accounts, an OIDC provider is required.

---

# 22. Associate IAM OIDC Provider

For an EKS cluster:

```bash
eksctl utils associate-iam-oidc-provider \
  --cluster kastro-cluster \
  --region us-east-1 \
  --approve
```

This allows Kubernetes service accounts to be associated with IAM roles through IAM Roles for Service Accounts (IRSA).

---

# 23. Check EBS CSI Driver

Check whether EBS CSI driver Pods are running:

```bash
kubectl get pods -n kube-system | grep ebs-csi
```

You may see components such as:

```text
ebs-csi-controller-...
ebs-csi-node-...
```

They should be in a healthy/running state.

---

# 24. Install EBS CSI Driver Using Helm

Install Helm if it is not already installed:

```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
```

Check Helm:

```bash
helm version
```

Add the EBS CSI driver repository:

```bash
helm repo add aws-ebs-csi-driver \
  https://kubernetes-sigs.github.io/aws-ebs-csi-driver
```

Update repositories:

```bash
helm repo update
```

Install:

```bash
helm install aws-ebs-csi-driver \
  aws-ebs-csi-driver/aws-ebs-csi-driver \
  --namespace kube-system \
  --create-namespace
```

Then verify:

```bash
kubectl get pods -n kube-system | grep ebs-csi
```

> The exact IAM and Helm configuration can vary by EKS setup. In particular, avoid assuming that attaching permissions directly to the worker-node IAM role is always the preferred production configuration; using a dedicated IAM role for the CSI controller is a common approach.

---

# 25. EBS Storage Architecture

After dynamic provisioning is configured:

```text
              Kubernetes
                   |
                   ↓
                  PVC
                   |
                   ↓
             StorageClass
                   |
                   ↓
            EBS CSI Driver
                   |
                   ↓
              AWS EBS
                   |
                   ↓
                  PV
                   |
                   ↓
                  Pod
```

The important point is that the application requests storage through a PVC rather than manually creating an EBS volume for every application.

---

# 26. Scenario B — Dynamic Provisioning with AWS EFS

AWS EFS is a managed shared file system.

It is useful when multiple Pods/nodes need shared access to the same filesystem.

A simplified architecture is:

```text
Pod 1 ──┐
Pod 2 ──┼──→ PVC → EFS CSI Driver → EFS
Pod 3 ──┘
```

EFS is commonly used for shared filesystem workloads.

---

# 27. Step 1 — Find the VPC

Example:

```bash
VPC_ID=$(aws ec2 describe-subnets \
  --subnet-ids subnet-06e1ce571dde7da18 \
  --query "Subnets[0].VpcId" \
  --output text)
```

Check:

```bash
echo $VPC_ID
```

---

# 28. Step 2 — Create Security Group

Create a security group for EFS:

```bash
aws ec2 create-security-group \
  --group-name efs-sg \
  --description "Security group for EFS" \
  --vpc-id $VPC_ID
```

Save the returned Security Group ID.

Example:

```text
sg-01ab1f1033a19b4c1
```

---

# 29. Allow NFS Traffic

EFS uses NFS.

The standard NFS port is:

```text
TCP 2049
```

For a lab:

```bash
aws ec2 authorize-security-group-ingress \
  --group-id <EFS-Security-Group-ID> \
  --protocol tcp \
  --port 2049 \
  --cidr 0.0.0.0/0
```

### Production recommendation

Do not normally expose NFS to:

```text
0.0.0.0/0
```

Instead, restrict access to the security group associated with the worker nodes or the appropriate application/network security boundary.

---

# 30. Step 3 — Create EFS File System

Create an EFS file system:

```bash
aws efs create-file-system \
  --creation-token kastro-efs \
  --performance-mode generalPurpose \
  --throughput-mode bursting
```

The response contains a File System ID.

Example:

```text
fs-03e917586b79c8118
```

Save the File System ID.

---

# 31. Step 4 — Create EFS Mount Targets

EFS requires mount targets in the VPC subnets/AZs from which clients need to access the file system.

Example:

```bash
aws efs create-mount-target \
  --file-system-id <FileSystemID> \
  --subnet-id <Subnet-ID-of-AZ1> \
  --security-groups <EFS-Security-Group-ID>
```

For another Availability Zone:

```bash
aws efs create-mount-target \
  --file-system-id <FileSystemID> \
  --subnet-id <Subnet-ID-of-AZ2> \
  --security-groups <EFS-Security-Group-ID>
```

Use subnets from **different Availability Zones** if you want the file system to be accessible across those AZs.

---

# 32. Verify EFS Mount Targets

```bash
aws efs describe-mount-targets \
  --file-system-id <FileSystemID>
```

You should see the mount targets and their associated subnets/AZs.

---

# 33. Install EFS CSI Driver

The EFS CSI driver allows Kubernetes to mount EFS into Pods.

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

Verify:

```bash
kubectl get pods -n kube-system | grep efs-csi
```

You should see EFS CSI driver components running.

---

# 34. EBS vs EFS

| Feature | EBS | EFS |
|---|---|---|
| Storage type | Block storage | Shared file storage |
| Typical access | RWO | RWX commonly used |
| Multiple nodes writing simultaneously | Generally no | Yes |
| Shared filesystem | No | Yes |
| Example use | Database, application data | Shared files |
| AWS service | Elastic Block Store | Elastic File System |
| CSI Driver | EBS CSI | EFS CSI |

Easy way to remember:

```text
EBS
 ↓
Block storage
 ↓
Usually one node at a time
```

```text
EFS
 ↓
Shared filesystem
 ↓
Multiple nodes can access
```

---

# 35. Complete Storage Flow

The complete Kubernetes storage flow can be remembered as:

```text
                    Application
                         |
                         ↓
                        Pod
                         |
                         ↓
                        PVC
                         |
                         ↓
                   StorageClass
                         |
                         ↓
                     CSI Driver
                         |
              ┌──────────┴──────────┐
              ↓                     ↓
            AWS EBS               AWS EFS
              ↓                     ↓
         Block Storage        Shared File System
```

---

# 36. Important Points to Remember

### PV

```text
PersistentVolume
↓
Represents storage available to Kubernetes
```

### PVC

```text
PersistentVolumeClaim
↓
Application's request for storage
```

### StorageClass

```text
Defines how storage should be dynamically provisioned
```

### CSI Driver

```text
Connects Kubernetes with storage systems
```

### Static Provisioning

```text
Admin creates PV
        ↓
PVC
        ↓
Pod
```

### Dynamic Provisioning

```text
PVC
 ↓
StorageClass
 ↓
CSI Driver
 ↓
Storage Backend
 ↓
PV
 ↓
Pod
```

### Access Modes

```text
RWO → ReadWriteOnce
ROX → ReadOnlyMany
RWX → ReadWriteMany
```

### Reclaim Policies

```text
Retain → Keep storage/PV
Delete → Delete dynamically provisioned storage/PV according to driver behavior
Recycle → Deprecated
```

### AWS

```text
EBS → Block storage → commonly RWO
EFS → Shared filesystem → commonly RWX
```

---

# 37. Final Memory Trick

Think of a hotel:

```text
PV
 ↓
Hotel room

PVC
 ↓
Customer booking the room

StorageClass
 ↓
Hotel room type/template

CSI Driver
 ↓
System that communicates with the hotel/storage provider

Pod
 ↓
Customer actually using the room
```

The most important flow is:

```text
Pod
 ↓
PVC
 ↓
PV
 ↓
Storage
```

For dynamic provisioning:

```text
Pod
 ↓
PVC
 ↓
StorageClass
 ↓
CSI Driver
 ↓
Storage Backend
 ↓
PV
```
