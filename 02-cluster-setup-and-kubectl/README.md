# 02 - Cluster Setup and kubectl

## 1. Docker Swarm vs Kubernetes Workflow

### Docker Swarm

```text
Cluster
   ↓
Nodes
   ↓
Containers
   ↓
Applications
```

### Kubernetes

```text
Cluster
   ↓
Nodes
   ↓
Pods
   ↓
Containers
   ↓
Applications
```

In Kubernetes, **Pods** are the smallest deployable units and contain one or more containers.

---

# 2. Creation of Minikube Cluster

## Environment

* Ubuntu 24.04
* AWS EC2
* Instance type: t2.medium
* Storage: 30 GB

## Install Docker

```bash
sudo apt update -y
sudo apt upgrade -y
sudo apt install curl wget apt-transport-https -y

sudo curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh
```

## Install Minikube

```bash
sudo curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64

sudo mv minikube-linux-amd64 /usr/local/bin/minikube

sudo chmod +x /usr/local/bin/minikube

sudo minikube version
```

## Install kubectl

```bash
sudo curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"

sudo curl -LO "https://dl.k8s.io/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl.sha256"

sudo echo "$(cat kubectl.sha256) kubectl" | sha256sum --check

sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
```

## Start Minikube

```bash
sudo minikube start --driver=docker --force
```

## Verify the Cluster

```bash
minikube status
```

```bash
kubectl get nodes
```

In a single-node Minikube cluster, the same node performs both control-plane and worker responsibilities.

Check namespaces:

```bash
kubectl get namespace
```

Check pods:

```bash
kubectl get pods
```

Check all resources:

```bash
kubectl get all
```

---

# 3. Kubernetes Configuration

Kubernetes client configuration is stored in:

```text
~/.kube/config
```

Check the `.kube` directory:

```bash
ls -la ~/.kube
```

View the configuration:

```bash
cat ~/.kube/config
```

The kubeconfig contains information that allows `kubectl` to connect to Kubernetes clusters.

---

# 4. Creation of EKS Cluster

Amazon EKS (Elastic Kubernetes Service) is AWS's managed Kubernetes service.

## EKS Management Host

An Ubuntu EC2 instance can be used as an EKS management host.

The main tools are:

| Tool      | Purpose                           |
| --------- | --------------------------------- |
| `kubectl` | Interact with Kubernetes clusters |
| `eksctl`  | Create and manage EKS clusters    |
| `aws`     | Interact with AWS services        |

---

## Step 1 - Create EKS Management Host

Launch:

* Ubuntu 24.04
* t2.micro
* 30 GB storage

The management host is used to run AWS CLI, eksctl and kubectl commands.

---

## Step 2 - IAM Role

Create an IAM role for the EC2 management host.

The role provides permissions required by the management host to interact with AWS services during EKS cluster creation.

In the lab, permissions for services such as:

* IAM
* VPC
* EC2
* CloudFormation

were used.

Attach the IAM role to the EC2 management host.

### AWS Services Involved

```text
IAM
 ↓
Provides permissions

VPC
 ↓
Provides networking

EC2
 ↓
Provides compute resources

CloudFormation
 ↓
Creates/manages AWS infrastructure

EKS
 ↓
Provides managed Kubernetes
```

---

## Alternative Authentication Method

Another approach is to create an IAM user and configure AWS CLI using access credentials.

```bash
aws configure
```

For real environments, prefer short-lived credentials or IAM roles instead of long-lived access keys whenever possible.

---

# 5. Install kubectl

Example installation used during the lab:

```bash
curl -o kubectl https://amazon-eks.s3.us-west-2.amazonaws.com/1.19.6/2021-01-05/bin/linux/amd64/kubectl

chmod +x ./kubectl

sudo mv ./kubectl /usr/local/bin

kubectl version --short --client
```

---

# 6. Install AWS CLI

```bash
sudo apt install unzip -y

curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"

unzip awscliv2.zip

sudo ./aws/install

aws --version
```

---

# 7. Install eksctl

```bash
curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp

sudo mv /tmp/eksctl /usr/local/bin

eksctl version
```

> The kubectl and eksctl commands above were used as part of the lab. Installation commands can change over time, so current official documentation should be checked when setting up a new environment.

---

# 8. Create EKS Cluster

### US East - N. Virginia

```bash
eksctl create cluster \
  --name kastro-cluster \
  --region us-east-1 \
  --node-type t2.medium \
  --zones us-east-1a,us-east-1b
```

### Mumbai

```bash
eksctl create cluster \
  --name kastro-cluster \
  --region ap-south-1 \
  --node-type t2.medium \
  --zones ap-south-1a,ap-south-1b
```

To specify the number of worker nodes:

```bash
eksctl create cluster \
  --name kastro-cluster \
  --region us-east-1 \
  --node-type t2.medium \
  --zones us-east-1a,us-east-1b \
  --nodes 3
```

Cluster creation takes several minutes.

---

# 9. Verify EKS Cluster

### List EKS clusters

```bash
aws eks list-clusters --region us-east-1
```

### Check worker nodes

```bash
kubectl get nodes
```

### Check cluster information

```bash
kubectl cluster-info
```

---

# 10. Delete EKS Cluster

After completing the practice, delete the cluster and unused resources to avoid unnecessary AWS charges.

```bash
eksctl delete cluster \
  --name kastro-cluster \
  --region us-east-1
```

---

# 11. kubectl Commands

## Nodes

### List nodes

```bash
kubectl get nodes
```

### Show additional node information

```bash
kubectl get nodes -o wide
```

### Detailed information about a node

```bash
kubectl describe node <NodeName>
```

---

## Namespaces

### List namespaces

```bash
kubectl get ns
```

or:

```bash
kubectl get namespaces
```

---

## Pods

### List pods

```bash
kubectl get pods
```

### List pods in a specific namespace

```bash
kubectl get pods -n <NamespaceName>
```

### Detailed information about a pod

```bash
kubectl describe pod <PodName>
```

### Detailed information about a pod in a namespace

```bash
kubectl describe pod <PodName> -n <NamespaceName>
```

---

## Resource Usage

### Node resource usage

```bash
kubectl top nodes
```

### Pod resource usage

```bash
kubectl top pods
```

> `kubectl top` requires the Kubernetes Metrics API to be available.

---

# 12. What I Practiced

* Docker Swarm vs Kubernetes workflow
* Kubernetes cluster structure
* Minikube cluster creation
* Docker as the Minikube driver
* kubectl installation
* Kubernetes kubeconfig
* EKS management host
* IAM role for an EC2 management host
* AWS CLI
* eksctl
* EKS cluster creation
* EKS cluster deletion
* Basic kubectl commands
* Nodes
* Pods
* Namespaces
* Cluster information
* Node and Pod resource usage
