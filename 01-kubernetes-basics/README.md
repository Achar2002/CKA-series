# Kubernetes Basics

## 1. What is Kubernetes?

Kubernetes (K8s) is an open-source container orchestration platform used to automate the deployment, scaling, load balancing, and management of containerized applications.

### Key Points

- Kubernetes is abbreviated as **K8s**.
- Kubernetes was originally developed by **Google**.
- Kubernetes was released as open source in **2014**.
- Kubernetes became part of the **Cloud Native Computing Foundation (CNCF)** in **2015**.
- The name Kubernetes comes from a Greek word meaning **helmsman** or **ship pilot**.

---

## 2. Why Do We Need Kubernetes?

Before Kubernetes, applications were commonly deployed using physical servers, virtual machines, or containers.

### Physical Servers

Applications running directly on physical servers could result in:

- Resource wastage
- Difficult scaling
- Difficult application management

### Virtual Machines

Virtual machines improved resource utilization and isolation.

However, VMs are relatively heavy because each VM requires its own operating system.

### Containers

Containers made applications:

- Lightweight
- Portable
- Faster to start
- Easier to package and deploy

However, managing hundreds or thousands of containers manually becomes difficult.

### Kubernetes

Kubernetes provides orchestration capabilities to automate the management of large numbers of containers.

---

## 3. Docker Swarm vs Kubernetes

Docker Swarm and Kubernetes are both container orchestration technologies.

Kubernetes provides a large ecosystem and is widely used for managing containerized workloads.

---

## 4. What is a Kubernetes Cluster?

A Kubernetes cluster is a group of machines that work together to run and manage containerized applications.

A Kubernetes cluster generally contains:

- **Control Plane**
- **Worker Nodes**

The control plane manages the cluster, while worker nodes run application workloads.

---

## 5. Kubernetes Cluster Setup

Kubernetes clusters can broadly be created in two ways.

### Self-Managed Kubernetes

The user is responsible for setting up and maintaining the cluster.

Examples:

- Minikube
- kubeadm
- KOps
- Kubespray
- Rancher
- K3s
- kind

### Managed Kubernetes

The cloud provider manages much of the Kubernetes infrastructure.

| Cloud Provider | Kubernetes Service |
|---|---|
| AWS | EKS |
| Google Cloud | GKE |
| Microsoft Azure | AKS |
| IBM Cloud | IKS |

---

## 6. Minikube

Minikube is a tool used to run Kubernetes locally or on a virtual machine for learning and development.

A single-node Minikube cluster can run both:

- Control plane components
- Worker workloads

### Environment Used

- Ubuntu 24.04
- 2 vCPU
- 30 GB storage
- Docker
- Minikube
- kubectl

---

## 7. Kubernetes Cluster Architecture

A Kubernetes cluster can be broadly divided into:

```text
                 Kubernetes Cluster
                         |
             +-----------+-----------+
             |                       |
        Control Plane            Worker Node
             |                       |
        API Server                Kubelet
        Scheduler             Container Runtime
        Controller Manager       kube-proxy
        etcd
