# Kubernetes - Container Orchestration Platform

## Table of Contents

- [1. Introduction to Container Orchestration](#1-introduction-to-container-orchestration)
- [2. What is Kubernetes](#2-what-is-kubernetes)
- [3. Installation and Setup](#3-installation-and-setup)
- [4. Kubernetes Core Concepts](#4-kubernetes-core-concepts)
- [5. Working with Kubernetes](#5-working-with-kubernetes)
- [6. Deploying a Python Flask App](#6-deploying-a-python-flask-app)
- [7. Troubleshooting Kubernetes](#7-troubleshooting-kubernetes)
- [8. Practice Exercises](#8-practice-exercises)

---

## 1. Introduction to Container Orchestration

### Why Container Orchestration is Needed

Containers solved the "it works on my machine" problem, but running containers in production introduces an entirely new set of challenges. When you have a single application running in a single container on a single server, everything is straightforward. But real-world applications rarely stay that simple.

Consider what happens as your application grows:

- **Scaling**: You need to run multiple instances of your container to handle traffic.
- **High Availability**: If a container crashes, something needs to restart it automatically.
- **Load Balancing**: Traffic must be distributed across container instances.
- **Service Discovery**: Containers need to find and communicate with each other.
- **Rolling Updates**: You need to deploy new versions without downtime.
- **Resource Management**: Containers must be scheduled on nodes with sufficient CPU and memory.
- **Secret Management**: Credentials and configuration must be distributed securely.
- **Storage Orchestration**: Persistent data must survive container restarts and rescheduling.

Container orchestration platforms automate all of these concerns, allowing you to declare the desired state of your system and letting the orchestrator make it so.

### What Happens When You Scale Beyond a Single VM

On a single VM, you can use Docker Compose to run multiple containers. But the moment you need more than one server, you face serious questions:

- Which server should a new container run on?
- What happens if a server goes down -- who restarts those containers?
- How do containers on Server A talk to containers on Server B?
- How do you perform a zero-downtime deployment across multiple servers?
- How do you share persistent storage across servers?

This is exactly the problem space that container orchestrators solve. They treat a cluster of machines as a single compute pool and handle scheduling, networking, and lifecycle management automatically.

### Kubernetes vs Docker Swarm vs Nomad

| Feature | Kubernetes | Docker Swarm | HashiCorp Nomad |
|---|---|---|---|
| **Complexity** | High (steep learning curve) | Low (easy to get started) | Medium |
| **Scalability** | Excellent (thousands of nodes) | Good (hundreds of nodes) | Excellent |
| **Community** | Massive, industry standard | Smaller, declining | Growing |
| **Setup** | Complex | Simple (built into Docker) | Moderate |
| **Service Discovery** | Built-in (DNS, Services) | Built-in | Consul integration |
| **Auto-scaling** | HPA, VPA, Cluster Autoscaler | Limited | Supported |
| **Networking** | CNI plugins (Calico, Flannel, etc.) | Overlay network | CNI plugins |
| **Storage** | CSI drivers, PV/PVC | Docker volumes | CSI support |
| **Use Case** | Production-grade, large-scale | Small-to-medium deployments | Multi-workload (containers, VMs, batch) |
| **Managed Offerings** | EKS, GKE, AKS, and many more | Limited | HCP Nomad |

**Bottom line**: Kubernetes has become the de facto industry standard for container orchestration. While Docker Swarm is simpler and Nomad is more flexible with workload types, Kubernetes dominates in terms of ecosystem, community, tooling, and job market demand. This guide focuses entirely on Kubernetes.

---

## 2. What is Kubernetes

Kubernetes (often abbreviated as **K8s** -- the "8" represents the eight letters between "K" and "s") is an open-source container orchestration platform that automates the deployment, scaling, and management of containerized applications.

### History: From Google Borg to Kubernetes

Kubernetes has its roots deep inside Google:

1. **2003-2004**: Google developed an internal cluster management system called **Borg**. Borg managed the deployment and scheduling of thousands of applications across millions of machines inside Google's data centers.

2. **2006**: Google introduced **Omega**, a next-generation cluster management system that improved upon Borg's scheduler with more flexible architecture.

3. **2014**: Google open-sourced a system called **Kubernetes**, drawing on 15+ years of experience running production workloads with Borg and Omega. The project was written in Go and released on GitHub.

4. **2015**: Kubernetes v1.0 was released, and Google donated the project to the newly formed **Cloud Native Computing Foundation (CNCF)**, part of the Linux Foundation.

5. **2017-Present**: Kubernetes became the industry standard for container orchestration. Every major cloud provider (AWS, Google Cloud, Azure) launched managed Kubernetes services. The CNCF ecosystem exploded with hundreds of projects building on top of Kubernetes.

### Kubernetes Architecture Deep Dive

A Kubernetes cluster consists of two main components: the **Control Plane** (master) and **Worker Nodes**.

```
+-----------------------------------------------------------------------+
|                        KUBERNETES CLUSTER                             |
|                                                                       |
|  +-----------------------------+    +------------------------------+  |
|  |       CONTROL PLANE         |    |        WORKER NODE 1         |  |
|  |                             |    |                              |  |
|  |  +-----------+              |    |  +----------+  +----------+  |  |
|  |  | API Server| <-------------------> | kubelet  |  |kube-proxy|  |  |
|  |  +-----------+              |    |  +----------+  +----------+  |  |
|  |       |                     |    |       |                      |  |
|  |  +-----------+              |    |  +----------+                |  |
|  |  |   etcd    |              |    |  |Container |                |  |
|  |  +-----------+              |    |  | Runtime  |                |  |
|  |       |                     |    |  +----------+                |  |
|  |  +-----------+              |    |       |                      |  |
|  |  | Scheduler |              |    |  +----+-----+  +----------+  |  |
|  |  +-----------+              |    |  |  Pod A   |  |  Pod B   |  |  |
|  |       |                     |    |  |+--------+|  |+--------+|  |  |
|  |  +-----------+              |    |  || Cont 1 ||  || Cont 1 ||  |  |
|  |  |Controller |              |    |  |+--------+|  |+--------+|  |  |
|  |  | Manager   |              |    |  +----------+  +----------+  |  |
|  |  +-----------+              |    +------------------------------+  |
|  +-----------------------------+                                      |
|                                     +------------------------------+  |
|                                     |        WORKER NODE 2         |  |
|                                     |                              |  |
|                                     |  +----------+  +----------+  |  |
|                                     |  | kubelet  |  |kube-proxy|  |  |
|                                     |  +----------+  +----------+  |  |
|                                     |       |                      |  |
|                                     |  +----------+                |  |
|                                     |  |Container |                |  |
|                                     |  | Runtime  |                |  |
|                                     |  +----------+                |  |
|                                     |       |                      |  |
|                                     |  +----+-----+  +----------+  |  |
|                                     |  |  Pod C   |  |  Pod D   |  |  |
|                                     |  |+--------+|  |+--------+|  |  |
|                                     |  || Cont 1 ||  || Cont 1 ||  |  |
|                                     |  |+--------+|  |+--------+|  |  |
|                                     |  +----------+  +----------+  |  |
|                                     +------------------------------+  |
+-----------------------------------------------------------------------+
```

#### Control Plane Components

The control plane makes global decisions about the cluster (for example, scheduling) and detects and responds to cluster events (for example, starting a new pod when a deployment's replica count is unsatisfied).

**API Server (`kube-apiserver`)**
- The front door to the Kubernetes cluster
- All communication (internal and external) goes through the API server
- Exposes a RESTful API over HTTPS
- Validates and processes API requests
- The only component that directly talks to etcd
- Handles authentication, authorization, and admission control

**etcd**
- A consistent, highly available key-value store
- Stores ALL cluster data: configuration, state, metadata
- Acts as the single source of truth for the cluster
- Only the API server communicates directly with etcd
- Should be backed up regularly in production environments
- Uses the Raft consensus algorithm for distributed consistency

**Scheduler (`kube-scheduler`)**
- Watches for newly created pods that have no node assigned
- Selects the best node for a pod to run on based on:
  - Resource requirements (CPU, memory)
  - Hardware/software constraints
  - Affinity and anti-affinity rules
  - Data locality
  - Taints and tolerations
- Does NOT run the pod -- it only decides WHERE the pod should run

**Controller Manager (`kube-controller-manager`)**
- Runs a collection of controller loops that regulate the state of the cluster
- Each controller watches the current state and works to move it toward the desired state
- Key controllers include:
  - **Node Controller**: Monitors node health, responds when nodes go down
  - **Replication Controller**: Ensures the correct number of pod replicas are running
  - **Endpoints Controller**: Populates the Endpoints object (joins Services and Pods)
  - **Service Account & Token Controllers**: Create default accounts and API access tokens

#### Worker Node Components

Worker nodes are the machines where your application containers actually run.

**kubelet**
- An agent that runs on every worker node
- Receives pod specifications (PodSpecs) from the API server
- Ensures the containers described in those PodSpecs are running and healthy
- Reports node and pod status back to the API server
- Does NOT manage containers that were not created by Kubernetes

**kube-proxy**
- A network proxy that runs on every worker node
- Implements Kubernetes Service networking
- Maintains network rules that allow communication to pods from inside or outside the cluster
- Uses iptables or IPVS rules to route traffic to the correct pod

**Container Runtime**
- The software responsible for actually running containers
- Kubernetes supports any runtime that implements the Container Runtime Interface (CRI)
- Common runtimes:
  - **containerd** (most common, default in modern Kubernetes)
  - **CRI-O** (lightweight, designed specifically for Kubernetes)
  - Docker Engine was supported via dockershim but was removed in Kubernetes 1.24

#### How Components Interact

Here is the flow when you run `kubectl apply -f deployment.yaml`:

1. `kubectl` sends the deployment manifest to the **API Server**
2. The **API Server** validates the request, authenticates/authorizes it, and stores the desired state in **etcd**
3. The **Controller Manager** (specifically the Deployment controller) detects the new Deployment, creates a ReplicaSet, which in turn detects it needs to create Pods
4. The **Scheduler** notices the new unassigned Pods and selects appropriate worker nodes
5. The **API Server** updates etcd with the node assignments
6. The **kubelet** on the assigned nodes picks up the new Pod specs
7. The **kubelet** instructs the **Container Runtime** to pull the image and start the containers
8. The **kube-proxy** on each node updates network rules so the pods are reachable
9. The **kubelet** continuously reports pod status back to the **API Server**

---

## 3. Installation and Setup

### kubectl Installation

`kubectl` is the command-line tool for interacting with Kubernetes clusters. You need it regardless of which cluster setup you choose.

#### macOS

```bash
# Using Homebrew (recommended)
brew install kubectl

# Or using curl
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/darwin/amd64/kubectl"
chmod +x kubectl
sudo mv kubectl /usr/local/bin/

# For Apple Silicon (M1/M2/M3/M4)
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/darwin/arm64/kubectl"
chmod +x kubectl
sudo mv kubectl /usr/local/bin/
```

#### Ubuntu/Debian

```bash
# Update package index and install prerequisites
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl

# Add Kubernetes signing key
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.31/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

# Add Kubernetes apt repository
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.31/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list

# Install kubectl
sudo apt-get update
sudo apt-get install -y kubectl
```

#### CentOS/RHEL

```bash
# Add Kubernetes yum repository
cat <<EOF | sudo tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://pkgs.k8s.io/core:/stable:/v1.31/rpm/
enabled=1
gpgcheck=1
gpgkey=https://pkgs.k8s.io/core:/stable:/v1.31/rpm/repodata/repomd.xml.key
EOF

# Install kubectl
sudo yum install -y kubectl
```

#### Windows

```powershell
# Using Chocolatey
choco install kubernetes-cli

# Or using curl (PowerShell)
curl.exe -LO "https://dl.k8s.io/release/v1.31.0/bin/windows/amd64/kubectl.exe"
# Move kubectl.exe to a directory in your PATH
```

#### Verifying kubectl

```bash
# Check version
kubectl version --client

# Expected output (version numbers may vary):
# Client Version: v1.31.x
# Kustomize Version: v5.x.x

# Enable shell autocompletion (bash)
echo 'source <(kubectl completion bash)' >> ~/.bashrc
source ~/.bashrc

# Enable shell autocompletion (zsh)
echo 'source <(kubectl completion zsh)' >> ~/.zshrc
source ~/.zshrc

# Create alias (optional but highly recommended)
echo 'alias k=kubectl' >> ~/.bashrc
echo 'complete -o default -F __start_kubectl k' >> ~/.bashrc
source ~/.bashrc
```

---

### Minikube (Local Development)

#### What is Minikube

Minikube is a tool that runs a single-node Kubernetes cluster on your local machine. It is the most popular way to learn Kubernetes and develop locally. Minikube creates a virtual machine (or uses a container driver like Docker) and sets up a complete Kubernetes environment inside it.

#### Installation

**macOS**

```bash
# Using Homebrew (recommended)
brew install minikube

# Or using curl (Intel)
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-darwin-amd64
sudo install minikube-darwin-amd64 /usr/local/bin/minikube

# Or using curl (Apple Silicon)
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-darwin-arm64
sudo install minikube-darwin-arm64 /usr/local/bin/minikube
```

**Linux**

```bash
# Using curl (amd64)
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube

# Using curl (arm64)
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-arm64
sudo install minikube-linux-arm64 /usr/local/bin/minikube

# Or using Debian package
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube_latest_amd64.deb
sudo dpkg -i minikube_latest_amd64.deb

# Or using RPM package
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-latest.x86_64.rpm
sudo rpm -Uvh minikube-latest.x86_64.rpm
```

**Windows**

```powershell
# Using Chocolatey
choco install minikube

# Or using winget
winget install minikube

# Or download the installer from:
# https://storage.googleapis.com/minikube/releases/latest/minikube-installer.exe
```

#### Starting a Cluster

```bash
# Start with default settings (uses Docker driver if available)
minikube start

# Start with specific driver
minikube start --driver=docker
minikube start --driver=virtualbox
minikube start --driver=hyperkit   # macOS only

# Start with specific Kubernetes version
minikube start --kubernetes-version=v1.31.0

# Start with custom resources
minikube start --cpus=4 --memory=8192 --disk-size=40g

# Start with a specific container runtime
minikube start --container-runtime=containerd

# Start with a named profile (for multiple clusters)
minikube start -p my-cluster
```

#### Cluster Management Commands

```bash
# Check cluster status
minikube status

# Stop the cluster (preserves state)
minikube stop

# Delete the cluster entirely
minikube delete

# Delete all profiles and clusters
minikube delete --all

# Pause Kubernetes (saves resources, keeps VM running)
minikube pause

# Unpause
minikube unpause

# SSH into the Minikube VM
minikube ssh

# View cluster IP
minikube ip

# Access a NodePort service URL
minikube service <service-name> --url

# Open a service directly in the browser
minikube service <service-name>

# View Minikube logs
minikube logs

# Check current profile
minikube profile list
```

#### Minikube Dashboard

Minikube comes with the Kubernetes Dashboard, a web-based UI for your cluster.

```bash
# Launch the dashboard (opens in browser)
minikube dashboard

# Get the dashboard URL without opening a browser
minikube dashboard --url
```

#### Minikube Addons

Minikube supports addons that extend cluster functionality.

```bash
# List all available addons
minikube addons list

# Enable addons
minikube addons enable ingress          # NGINX Ingress Controller
minikube addons enable metrics-server   # Resource metrics for HPA
minikube addons enable dashboard        # Kubernetes Dashboard
minikube addons enable registry         # Container registry
minikube addons enable storage-provisioner  # Dynamic volume provisioning

# Disable an addon
minikube addons disable ingress

# Open an addon (if it provides a UI)
minikube addons open dashboard
```

#### Using Local Docker Images with Minikube

To use locally built Docker images with Minikube (without pushing to a registry):

```bash
# Point your Docker CLI to Minikube's Docker daemon
eval $(minikube docker-env)

# Now build your image -- it will be available inside Minikube
docker build -t my-app:v1 .

# In your pod/deployment YAML, set:
# imagePullPolicy: Never
# or
# imagePullPolicy: IfNotPresent

# To revert back to your host Docker daemon
eval $(minikube docker-env -u)
```

---

### Kind (Kubernetes in Docker) - Alternative

Kind (Kubernetes IN Docker) runs Kubernetes clusters using Docker containers as nodes. It is popular for CI/CD pipelines and testing.

#### Installation

```bash
# macOS (Homebrew)
brew install kind

# Linux / macOS (using Go)
go install sigs.k8s.io/kind@latest

# Linux (binary)
curl -Lo ./kind https://kind.sigs.k8s.io/dl/latest/kind-linux-amd64
chmod +x ./kind
sudo mv ./kind /usr/local/bin/kind

# Windows (Chocolatey)
choco install kind
```

#### Creating a Cluster

```bash
# Create a cluster with default settings
kind create cluster

# Create a named cluster
kind create cluster --name my-cluster

# Create a cluster with a specific Kubernetes version
kind create cluster --image kindest/node:v1.31.0

# Delete a cluster
kind delete cluster --name my-cluster

# List clusters
kind get clusters

# Export kubeconfig
kind get kubeconfig --name my-cluster
```

#### Multi-Node Clusters with Kind

Create a configuration file for a multi-node cluster:

```yaml
# kind-multi-node.yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
  - role: control-plane
  - role: worker
  - role: worker
  - role: worker
```

```bash
# Create the multi-node cluster
kind create cluster --config kind-multi-node.yaml --name multi-node

# Verify the nodes
kubectl get nodes
```

Loading local Docker images into Kind:

```bash
# Load a local image into the Kind cluster
kind load docker-image my-app:v1 --name my-cluster
```

---

### Cloud Kubernetes (Overview)

#### Amazon EKS (Elastic Kubernetes Service)

- Managed Kubernetes on AWS
- AWS manages the control plane (API server, etcd, scheduler, controller manager)
- You manage the worker nodes (EC2 instances or Fargate serverless)
- Integrates with AWS services: ALB, IAM, CloudWatch, ECR
- Setup via `eksctl`, AWS CLI, CloudFormation, or Terraform
- Pricing: $0.10/hour for the control plane + node costs

```bash
# Quick EKS cluster creation with eksctl
eksctl create cluster \
  --name my-cluster \
  --region us-west-2 \
  --nodes 3 \
  --node-type t3.medium
```

#### Google GKE (Google Kubernetes Engine)

- Managed Kubernetes on Google Cloud
- Most mature managed Kubernetes offering (Google created Kubernetes)
- GKE Autopilot mode: Google manages everything including nodes
- GKE Standard mode: you manage node pools
- Integrates with Google Cloud services: Cloud Load Balancing, Cloud IAM, Cloud Monitoring
- Pricing: GKE Standard control plane is free; Autopilot charges per pod resource usage

```bash
# Quick GKE cluster creation
gcloud container clusters create my-cluster \
  --zone us-central1-a \
  --num-nodes 3 \
  --machine-type e2-medium
```

#### Azure AKS (Azure Kubernetes Service)

- Managed Kubernetes on Microsoft Azure
- Free control plane (you only pay for worker nodes)
- Integrates with Azure services: Azure Active Directory, Azure Monitor, ACR
- Supports Windows and Linux node pools
- Setup via Azure CLI, Azure Portal, or Terraform

```bash
# Quick AKS cluster creation
az aks create \
  --resource-group myResourceGroup \
  --name my-cluster \
  --node-count 3 \
  --node-vm-size Standard_DS2_v2 \
  --generate-ssh-keys
```

---

## 4. Kubernetes Core Concepts

### Pods

#### What is a Pod

A Pod is the smallest deployable unit in Kubernetes. It represents a single instance of a running process in your cluster.

Key characteristics:
- A Pod encapsulates one or more containers that share the same network namespace (same IP address and port space)
- Containers in the same Pod can communicate via `localhost`
- A Pod shares storage volumes among its containers
- Pods are ephemeral -- they are created, they run, and they die. They are never "healed" -- a new pod is created instead
- Each Pod gets its own unique IP address within the cluster

#### Pod Lifecycle

A Pod moves through several phases during its lifetime:

| Phase | Description |
|---|---|
| **Pending** | Pod accepted by the cluster, but one or more containers are not yet running. This includes time spent downloading images. |
| **Running** | Pod bound to a node, all containers created, at least one is running or starting/restarting. |
| **Succeeded** | All containers terminated successfully and will not be restarted. |
| **Failed** | All containers terminated, at least one terminated in failure. |
| **Unknown** | Pod state cannot be determined, usually due to communication error with the node. |

#### Multi-Container Pods (Sidecar Pattern)

While most Pods run a single container, there are valid reasons to run multiple containers in one Pod:

- **Sidecar**: A helper container that enhances the main container (for example, a log shipping agent or a proxy)
- **Ambassador**: A proxy container that simplifies access to external services
- **Adapter**: A container that standardizes output from the main container

```yaml
# multi-container-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: web-with-sidecar
  labels:
    app: web
spec:
  containers:
    # Main application container
    - name: web-app
      image: nginx:1.27
      ports:
        - containerPort: 80
      volumeMounts:
        - name: shared-logs
          mountPath: /var/log/nginx

    # Sidecar container that ships logs
    - name: log-shipper
      image: busybox:1.36
      command: ["sh", "-c", "tail -f /var/log/nginx/access.log"]
      volumeMounts:
        - name: shared-logs
          mountPath: /var/log/nginx

  volumes:
    - name: shared-logs
      emptyDir: {}
```

#### Pod YAML Spec -- Full Example

```yaml
# pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-app
  namespace: default
  labels:
    app: my-app
    environment: development
    version: v1.0.0
  annotations:
    description: "My application pod"
spec:
  containers:
    - name: my-app
      image: nginx:1.27
      ports:
        - containerPort: 80
          protocol: TCP
      env:
        - name: APP_ENV
          value: "development"
        - name: APP_PORT
          value: "80"
      resources:
        requests:
          cpu: "100m"
          memory: "128Mi"
        limits:
          cpu: "250m"
          memory: "256Mi"
      livenessProbe:
        httpGet:
          path: /
          port: 80
        initialDelaySeconds: 10
        periodSeconds: 5
      readinessProbe:
        httpGet:
          path: /
          port: 80
        initialDelaySeconds: 5
        periodSeconds: 3
  restartPolicy: Always
```

#### Running and Managing Pods

```bash
# Run a pod imperatively (quick testing)
kubectl run nginx --image=nginx:1.27 --port=80

# Run a pod with labels
kubectl run nginx --image=nginx:1.27 --labels="app=nginx,env=dev"

# Create a pod from YAML
kubectl apply -f pod.yaml

# View all pods in current namespace
kubectl get pods

# View pods with more detail
kubectl get pods -o wide

# View pods in all namespaces
kubectl get pods --all-namespaces
# or
kubectl get pods -A

# View pod details
kubectl describe pod my-app

# View pod YAML
kubectl get pod my-app -o yaml

# View pod as JSON
kubectl get pod my-app -o json

# Watch pods in real-time
kubectl get pods -w

# Filter pods by label
kubectl get pods -l app=my-app
kubectl get pods -l "environment in (production, staging)"
```

#### Pod Logs

```bash
# View pod logs
kubectl logs my-app

# View logs for a specific container in a multi-container pod
kubectl logs my-app -c log-shipper

# Follow logs in real-time (like tail -f)
kubectl logs -f my-app

# View last N lines
kubectl logs --tail=100 my-app

# View logs from the previous instance of a crashed container
kubectl logs my-app --previous

# View logs with timestamps
kubectl logs my-app --timestamps=true

# View logs from the last hour
kubectl logs my-app --since=1h

# View logs from all pods with a specific label
kubectl logs -l app=my-app --all-containers
```

#### Exec Into Pods

```bash
# Execute a command in a running pod
kubectl exec my-app -- ls /

# Open an interactive shell
kubectl exec -it my-app -- /bin/bash

# Open a shell in a specific container (multi-container pod)
kubectl exec -it my-app -c log-shipper -- /bin/sh

# Run a one-off command
kubectl exec my-app -- cat /etc/nginx/nginx.conf

# Copy files to/from a pod
kubectl cp my-app:/var/log/nginx/access.log ./access.log
kubectl cp ./local-file.txt my-app:/tmp/file.txt
```

#### Deleting Pods

```bash
# Delete a pod
kubectl delete pod my-app

# Delete a pod immediately (force delete)
kubectl delete pod my-app --grace-period=0 --force

# Delete all pods with a specific label
kubectl delete pods -l app=my-app

# Delete from YAML file
kubectl delete -f pod.yaml
```

---

### ReplicaSets

#### What and Why

A ReplicaSet ensures that a specified number of identical pod replicas are running at all times. If a pod crashes, the ReplicaSet controller creates a new one to replace it. If there are too many pods, it terminates the extras.

You rarely create ReplicaSets directly. Instead, you use **Deployments**, which manage ReplicaSets for you and provide additional features like rolling updates. However, understanding ReplicaSets is important for understanding how Kubernetes works.

#### YAML Example

```yaml
# replicaset.yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: nginx-replicaset
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
        - name: nginx
          image: nginx:1.27
          ports:
            - containerPort: 80
          resources:
            requests:
              cpu: "100m"
              memory: "128Mi"
            limits:
              cpu: "250m"
              memory: "256Mi"
```

#### Managing ReplicaSets

```bash
# Create the ReplicaSet
kubectl apply -f replicaset.yaml

# View ReplicaSets
kubectl get replicasets
kubectl get rs

# Describe a ReplicaSet
kubectl describe rs nginx-replicaset

# Scale the ReplicaSet
kubectl scale rs nginx-replicaset --replicas=5

# Delete the ReplicaSet (also deletes its pods)
kubectl delete rs nginx-replicaset

# Delete the ReplicaSet but keep its pods (orphan)
kubectl delete rs nginx-replicaset --cascade=orphan
```

---

### Deployments

#### What is a Deployment

A Deployment provides declarative updates for Pods and ReplicaSets. It is the most commonly used Kubernetes resource for running stateless applications. A Deployment manages ReplicaSets, which in turn manage Pods.

Key features:
- Declarative updates (you describe the desired state)
- Rolling updates with zero downtime
- Rollback to previous versions
- Scaling up and down
- Pause and resume updates

#### Deployment YAML -- Full Example

```yaml
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  namespace: default
  labels:
    app: nginx
    environment: production
  annotations:
    kubernetes.io/change-cause: "Initial deployment of nginx 1.27"
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1     # At most 1 pod can be unavailable during update
      maxSurge: 1            # At most 1 extra pod can be created during update
  template:
    metadata:
      labels:
        app: nginx
        version: v1.0.0
    spec:
      containers:
        - name: nginx
          image: nginx:1.27
          ports:
            - containerPort: 80
              protocol: TCP
          env:
            - name: ENVIRONMENT
              value: "production"
          resources:
            requests:
              cpu: "100m"
              memory: "128Mi"
            limits:
              cpu: "500m"
              memory: "256Mi"
          livenessProbe:
            httpGet:
              path: /
              port: 80
            initialDelaySeconds: 10
            periodSeconds: 10
            failureThreshold: 3
          readinessProbe:
            httpGet:
              path: /
              port: 80
            initialDelaySeconds: 5
            periodSeconds: 5
            failureThreshold: 3
      restartPolicy: Always
```

#### Deployment Strategies

**RollingUpdate (Default)**

Gradually replaces old pods with new ones. At no point are all pods unavailable.

```yaml
spec:
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1    # max pods that can be unavailable during update
      maxSurge: 1           # max pods that can be created above desired count
```

**Recreate**

Terminates all existing pods before creating new ones. Causes downtime but ensures no two versions run simultaneously.

```yaml
spec:
  strategy:
    type: Recreate
```

#### Creating and Managing Deployments

```bash
# Create a deployment imperatively
kubectl create deployment nginx --image=nginx:1.27 --replicas=3

# Create from YAML
kubectl apply -f deployment.yaml

# View deployments
kubectl get deployments
kubectl get deploy

# View deployment details
kubectl describe deployment nginx-deployment

# View the ReplicaSets created by a deployment
kubectl get rs -l app=nginx

# View the pods created by a deployment
kubectl get pods -l app=nginx
```

#### Scaling Deployments

```bash
# Scale to 5 replicas
kubectl scale deployment nginx-deployment --replicas=5

# Autoscale based on CPU usage (requires metrics-server)
kubectl autoscale deployment nginx-deployment --min=3 --max=10 --cpu-percent=80

# View Horizontal Pod Autoscaler status
kubectl get hpa
```

#### Rolling Updates and Rollbacks

```bash
# Update the image (triggers a rolling update)
kubectl set image deployment/nginx-deployment nginx=nginx:1.27.3

# Or edit the deployment directly
kubectl edit deployment nginx-deployment

# Or update via YAML and apply
kubectl apply -f deployment.yaml

# Check rollout status
kubectl rollout status deployment/nginx-deployment

# View rollout history
kubectl rollout history deployment/nginx-deployment

# View details of a specific revision
kubectl rollout history deployment/nginx-deployment --revision=2

# Rollback to the previous version
kubectl rollout undo deployment/nginx-deployment

# Rollback to a specific revision
kubectl rollout undo deployment/nginx-deployment --to-revision=1

# Pause a rollout (for canary-like testing)
kubectl rollout pause deployment/nginx-deployment

# Resume a paused rollout
kubectl rollout resume deployment/nginx-deployment

# Restart all pods in a deployment (rolling restart)
kubectl rollout restart deployment/nginx-deployment
```

---

### Services

A Service is an abstraction that defines a logical set of Pods and a policy by which to access them. Services provide stable networking for Pods, which have ephemeral IP addresses.

Why you need Services:
- Pods are ephemeral and their IP addresses change
- Services provide a stable IP address and DNS name
- Services load-balance traffic across a set of Pods
- Services enable loose coupling between microservices

#### Service Types

**ClusterIP (Default)**

Exposes the Service on an internal IP address reachable only within the cluster. Use this for internal service-to-service communication.

```yaml
# service-clusterip.yaml
apiVersion: v1
kind: Service
metadata:
  name: my-app-service
spec:
  type: ClusterIP
  selector:
    app: my-app
  ports:
    - protocol: TCP
      port: 80          # Port the service listens on
      targetPort: 8080   # Port on the container
```

**NodePort**

Exposes the Service on a static port on each node's IP. Accessible from outside the cluster via `<NodeIP>:<NodePort>`.

```yaml
# service-nodeport.yaml
apiVersion: v1
kind: Service
metadata:
  name: my-app-nodeport
spec:
  type: NodePort
  selector:
    app: my-app
  ports:
    - protocol: TCP
      port: 80           # ClusterIP port
      targetPort: 8080    # Container port
      nodePort: 30080     # External port (range: 30000-32767)
```

**LoadBalancer**

Exposes the Service externally using a cloud provider's load balancer. Only works with cloud providers (AWS, GCP, Azure) or MetalLB on bare metal.

```yaml
# service-loadbalancer.yaml
apiVersion: v1
kind: Service
metadata:
  name: my-app-lb
spec:
  type: LoadBalancer
  selector:
    app: my-app
  ports:
    - protocol: TCP
      port: 80
      targetPort: 8080
```

**ExternalName**

Maps a Service to an external DNS name. No proxying -- just a CNAME redirect at the DNS level.

```yaml
# service-externalname.yaml
apiVersion: v1
kind: Service
metadata:
  name: external-db
spec:
  type: ExternalName
  externalName: database.example.com
```

#### Service Discovery

Kubernetes provides two methods for service discovery:

**DNS (Recommended)**

Every Service gets a DNS entry in the format:
```
<service-name>.<namespace>.svc.cluster.local
```

For example, a service called `my-app-service` in the `default` namespace is reachable at:
```
my-app-service.default.svc.cluster.local
```

Within the same namespace, you can simply use:
```
my-app-service
```

**Environment Variables**

When a Pod starts, Kubernetes injects environment variables for each active Service:
```
MY_APP_SERVICE_SERVICE_HOST=10.96.0.1
MY_APP_SERVICE_SERVICE_PORT=80
```

#### Managing Services

```bash
# Create a service imperatively
kubectl expose deployment nginx-deployment --port=80 --target-port=80 --type=ClusterIP

# Create from YAML
kubectl apply -f service-clusterip.yaml

# View services
kubectl get services
kubectl get svc

# View service details
kubectl describe svc my-app-service

# View service endpoints (which pods are behind the service)
kubectl get endpoints my-app-service

# Access a ClusterIP service from outside the cluster using port-forward
kubectl port-forward svc/my-app-service 8080:80

# Delete a service
kubectl delete svc my-app-service
```

---

### Namespaces

#### What and Why

Namespaces provide a mechanism for isolating groups of resources within a single cluster. They are useful for:

- Separating environments (dev, staging, production)
- Multi-tenant clusters (different teams or projects)
- Applying resource quotas per team or environment
- Controlling access with RBAC policies

Kubernetes comes with several default namespaces:

| Namespace | Purpose |
|---|---|
| `default` | Default namespace for resources with no namespace specified |
| `kube-system` | For resources created by the Kubernetes system |
| `kube-public` | Readable by all users, used for public resources |
| `kube-node-lease` | Holds Lease objects for node heartbeats |

#### Creating and Using Namespaces

```bash
# Create a namespace imperatively
kubectl create namespace development

# Create from YAML
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Namespace
metadata:
  name: development
  labels:
    environment: development
EOF

# List namespaces
kubectl get namespaces
kubectl get ns

# View resources in a specific namespace
kubectl get pods -n development
kubectl get all -n development

# Set default namespace for current context (avoid typing -n every time)
kubectl config set-context --current --namespace=development

# Verify current namespace
kubectl config view --minify --output 'jsonpath={..namespace}'

# Delete a namespace (deletes ALL resources in it)
kubectl delete namespace development
```

#### Resource Quotas per Namespace

```yaml
# resource-quota.yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: dev-quota
  namespace: development
spec:
  hard:
    requests.cpu: "4"
    requests.memory: "8Gi"
    limits.cpu: "8"
    limits.memory: "16Gi"
    pods: "20"
    services: "10"
    configmaps: "20"
    secrets: "20"
    persistentvolumeclaims: "10"
```

```bash
# Apply the quota
kubectl apply -f resource-quota.yaml

# View quota usage
kubectl describe resourcequota dev-quota -n development
```

#### LimitRange (Default Limits per Namespace)

```yaml
# limit-range.yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: default-limits
  namespace: development
spec:
  limits:
    - default:
        cpu: "500m"
        memory: "256Mi"
      defaultRequest:
        cpu: "100m"
        memory: "128Mi"
      type: Container
```

```bash
kubectl apply -f limit-range.yaml
```

---

### ConfigMaps and Secrets

#### ConfigMaps

ConfigMaps store non-confidential configuration data as key-value pairs. They decouple configuration from container images, making applications portable.

**Creating ConfigMaps**

```bash
# From literal values
kubectl create configmap app-config \
  --from-literal=APP_ENV=production \
  --from-literal=APP_DEBUG=false \
  --from-literal=APP_PORT=8080

# From a file
echo "database.host=db.example.com" > db.properties
echo "database.port=5432" >> db.properties
kubectl create configmap db-config --from-file=db.properties

# From an env file
cat > app.env << EOF
APP_ENV=production
APP_DEBUG=false
APP_PORT=8080
EOF
kubectl create configmap app-config --from-env-file=app.env

# From YAML
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  APP_ENV: "production"
  APP_DEBUG: "false"
  APP_PORT: "8080"
  nginx.conf: |
    server {
        listen 80;
        server_name localhost;
        location / {
            root /usr/share/nginx/html;
            index index.html;
        }
    }
EOF
```

**Viewing ConfigMaps**

```bash
kubectl get configmaps
kubectl get cm
kubectl describe configmap app-config
kubectl get configmap app-config -o yaml
```

#### Secrets

Secrets are similar to ConfigMaps but are designed for sensitive data (passwords, tokens, keys). Secret values are base64-encoded (not encrypted by default -- you need additional tooling for encryption at rest).

**Creating Secrets**

```bash
# From literal values
kubectl create secret generic db-secret \
  --from-literal=DB_USER=admin \
  --from-literal=DB_PASSWORD=s3cur3p@ssw0rd

# From files
echo -n 'admin' > username.txt
echo -n 's3cur3p@ssw0rd' > password.txt
kubectl create secret generic db-secret \
  --from-file=DB_USER=username.txt \
  --from-file=DB_PASSWORD=password.txt

# From YAML (values must be base64-encoded)
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Secret
metadata:
  name: db-secret
type: Opaque
data:
  DB_USER: YWRtaW4=           # echo -n 'admin' | base64
  DB_PASSWORD: czNjdXIzcEBzc3cwcmQ=  # echo -n 's3cur3p@ssw0rd' | base64
EOF

# Using stringData (plain text, Kubernetes encodes it for you)
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Secret
metadata:
  name: db-secret
type: Opaque
stringData:
  DB_USER: admin
  DB_PASSWORD: s3cur3p@ssw0rd
EOF

# Create a TLS secret
kubectl create secret tls tls-secret \
  --cert=tls.crt \
  --key=tls.key

# Create a Docker registry secret
kubectl create secret docker-registry regcred \
  --docker-server=https://index.docker.io/v1/ \
  --docker-username=myuser \
  --docker-password=mypassword \
  --docker-email=user@example.com
```

#### Using ConfigMaps and Secrets in Pods

**As Environment Variables**

```yaml
# pod-with-config.yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-with-config
spec:
  containers:
    - name: app
      image: nginx:1.27
      env:
        # Individual keys from ConfigMap
        - name: APP_ENV
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: APP_ENV
        # Individual keys from Secret
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: db-secret
              key: DB_PASSWORD
      # All keys from ConfigMap as env vars
      envFrom:
        - configMapRef:
            name: app-config
        - secretRef:
            name: db-secret
```

**As Volume Mounts**

```yaml
# pod-with-config-volume.yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-with-config-volume
spec:
  containers:
    - name: app
      image: nginx:1.27
      volumeMounts:
        - name: config-volume
          mountPath: /etc/config
          readOnly: true
        - name: secret-volume
          mountPath: /etc/secrets
          readOnly: true
  volumes:
    - name: config-volume
      configMap:
        name: app-config
        items:
          - key: nginx.conf
            path: nginx.conf
    - name: secret-volume
      secret:
        secretName: db-secret
        defaultMode: 0400    # File permission: read-only for owner
```

When mounted as a volume, each key in the ConfigMap/Secret becomes a file, and the value becomes the file content. Changes to ConfigMaps/Secrets are automatically reflected in the mounted volume (with a short delay), without restarting the pod.

---

### Volumes and Persistent Storage

Containers are ephemeral by default -- any data written inside a container is lost when it restarts. Kubernetes volumes solve this problem.

#### Volume Types

**emptyDir**

A temporary directory that is created when a Pod is assigned to a node and exists as long as the Pod runs. Useful for scratch space and sharing data between containers in the same pod.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-with-emptydir
spec:
  containers:
    - name: writer
      image: busybox:1.36
      command: ["sh", "-c", "echo 'Hello' > /data/hello.txt && sleep 3600"]
      volumeMounts:
        - name: shared-data
          mountPath: /data
    - name: reader
      image: busybox:1.36
      command: ["sh", "-c", "cat /data/hello.txt && sleep 3600"]
      volumeMounts:
        - name: shared-data
          mountPath: /data
  volumes:
    - name: shared-data
      emptyDir: {}
```

**hostPath**

Mounts a file or directory from the host node's filesystem. Useful for development but generally discouraged in production because it ties the pod to a specific node.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-with-hostpath
spec:
  containers:
    - name: app
      image: nginx:1.27
      volumeMounts:
        - name: host-data
          mountPath: /usr/share/nginx/html
  volumes:
    - name: host-data
      hostPath:
        path: /data/web
        type: DirectoryOrCreate
```

#### PersistentVolumes and PersistentVolumeClaims

The PersistentVolume (PV) and PersistentVolumeClaim (PVC) system decouples storage provisioning from consumption:

- **PersistentVolume (PV)**: A piece of storage in the cluster provisioned by an administrator or dynamically via a StorageClass
- **PersistentVolumeClaim (PVC)**: A request for storage by a user. Pods use PVCs to consume PV resources.

```yaml
# persistent-volume.yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: my-pv
spec:
  capacity:
    storage: 10Gi
  volumeMode: Filesystem
  accessModes:
    - ReadWriteOnce    # Can be mounted read-write by a single node
  persistentVolumeReclaimPolicy: Retain   # Keep data after PVC is deleted
  storageClassName: standard
  hostPath:
    path: /mnt/data
```

```yaml
# persistent-volume-claim.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
  storageClassName: standard
```

```yaml
# pod-with-pvc.yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-with-pvc
spec:
  containers:
    - name: app
      image: nginx:1.27
      volumeMounts:
        - name: persistent-storage
          mountPath: /usr/share/nginx/html
  volumes:
    - name: persistent-storage
      persistentVolumeClaim:
        claimName: my-pvc
```

**Access Modes:**
| Mode | Short | Description |
|---|---|---|
| ReadWriteOnce | RWO | Mounted as read-write by a single node |
| ReadOnlyMany | ROX | Mounted as read-only by many nodes |
| ReadWriteMany | RWX | Mounted as read-write by many nodes |
| ReadWriteOncePod | RWOP | Mounted as read-write by a single pod |

**Reclaim Policies:**
| Policy | Description |
|---|---|
| Retain | Keep the PV and its data after PVC is deleted (manual cleanup) |
| Delete | Delete the PV and its underlying storage when PVC is deleted |
| Recycle | (Deprecated) Basic scrub (`rm -rf /thevolume/*`) |

#### StorageClasses

StorageClasses provide a way to describe "classes" of storage and enable dynamic provisioning of PersistentVolumes.

```yaml
# storage-class.yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-ssd
provisioner: kubernetes.io/aws-ebs   # Cloud-specific provisioner
parameters:
  type: gp3
  fsType: ext4
reclaimPolicy: Delete
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true
```

```yaml
# pvc-with-storageclass.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: fast-storage
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 20Gi
  storageClassName: fast-ssd   # References the StorageClass above
```

```bash
# View storage classes
kubectl get storageclass
kubectl get sc

# View persistent volumes
kubectl get pv

# View persistent volume claims
kubectl get pvc

# Describe a PVC
kubectl describe pvc my-pvc
```

---

### Ingress

#### What is Ingress

Ingress exposes HTTP and HTTPS routes from outside the cluster to Services within the cluster. It provides:

- URL-based routing (path-based and host-based)
- SSL/TLS termination
- Name-based virtual hosting
- Load balancing

An Ingress resource on its own does nothing. You need an **Ingress Controller** (such as NGINX Ingress Controller) running in the cluster to fulfill Ingress rules.

#### Ingress Controllers

The most common Ingress Controller is the **NGINX Ingress Controller**. On Minikube:

```bash
# Enable the NGINX Ingress Controller on Minikube
minikube addons enable ingress

# Verify it is running
kubectl get pods -n ingress-nginx
```

For production clusters, you install an Ingress Controller via Helm or manifests:

```bash
# Install NGINX Ingress Controller via Helm
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
helm install ingress-nginx ingress-nginx/ingress-nginx
```

#### Ingress YAML Examples

**Path-Based Routing**

Route requests to different services based on the URL path:

```yaml
# ingress-path.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: path-based-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
    - host: myapp.example.com
      http:
        paths:
          - path: /api
            pathType: Prefix
            backend:
              service:
                name: api-service
                port:
                  number: 80
          - path: /web
            pathType: Prefix
            backend:
              service:
                name: web-service
                port:
                  number: 80
          - path: /
            pathType: Prefix
            backend:
              service:
                name: frontend-service
                port:
                  number: 80
```

**Host-Based Routing**

Route requests to different services based on the hostname:

```yaml
# ingress-host.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: host-based-ingress
spec:
  ingressClassName: nginx
  rules:
    - host: api.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: api-service
                port:
                  number: 80
    - host: app.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: web-service
                port:
                  number: 80
```

**TLS Termination**

```yaml
# ingress-tls.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: tls-ingress
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - myapp.example.com
      secretName: myapp-tls-secret    # Must be a TLS-type Secret
  rules:
    - host: myapp.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: web-service
                port:
                  number: 80
```

Create the TLS secret:

```bash
# Self-signed certificate (for testing only)
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout tls.key -out tls.crt \
  -subj "/CN=myapp.example.com"

kubectl create secret tls myapp-tls-secret --cert=tls.crt --key=tls.key
```

#### Managing Ingress

```bash
# Create Ingress
kubectl apply -f ingress-path.yaml

# View Ingress resources
kubectl get ingress
kubectl get ing

# Describe Ingress
kubectl describe ingress path-based-ingress

# Delete Ingress
kubectl delete ingress path-based-ingress
```

---

## 5. Working with Kubernetes

### kubectl Essential Commands

#### Getting Information

```bash
# Cluster information
kubectl cluster-info
kubectl version
kubectl api-resources          # List all resource types
kubectl api-versions           # List all API versions

# Get resources (works with any resource type)
kubectl get pods
kubectl get deployments
kubectl get services
kubectl get nodes
kubectl get all                # Get most common resources
kubectl get all -A             # All resources in all namespaces

# Output formats
kubectl get pods -o wide       # Extended output
kubectl get pods -o yaml       # YAML format
kubectl get pods -o json       # JSON format
kubectl get pods -o name       # Just resource names
kubectl get pods -o custom-columns="NAME:.metadata.name,STATUS:.status.phase"

# Sorting
kubectl get pods --sort-by=.metadata.creationTimestamp
kubectl get pods --sort-by=.status.phase

# Filtering by labels
kubectl get pods -l app=nginx
kubectl get pods -l "app in (nginx, web)"
kubectl get pods -l app=nginx,environment=production

# Field selectors
kubectl get pods --field-selector status.phase=Running
kubectl get pods --field-selector metadata.name=my-pod
```

#### Describing Resources

```bash
# Detailed info about a specific resource
kubectl describe pod my-pod
kubectl describe deployment my-deployment
kubectl describe node my-node
kubectl describe svc my-service

# Events in the cluster
kubectl get events
kubectl get events --sort-by=.lastTimestamp
kubectl get events -n kube-system
```

#### Creating and Applying

```bash
# Create resources imperatively
kubectl create namespace my-namespace
kubectl create deployment nginx --image=nginx:1.27
kubectl create configmap my-config --from-literal=key=value
kubectl create secret generic my-secret --from-literal=password=secret

# Apply resources declaratively (preferred)
kubectl apply -f manifest.yaml
kubectl apply -f directory/               # Apply all YAML in a directory
kubectl apply -f https://example.com/manifest.yaml  # Apply from URL
kubectl apply -k ./kustomization/         # Apply with Kustomize

# Dry run (preview without actually applying)
kubectl apply -f manifest.yaml --dry-run=client
kubectl apply -f manifest.yaml --dry-run=server   # Server-side validation

# Generate YAML without creating (useful for creating templates)
kubectl create deployment nginx --image=nginx:1.27 --dry-run=client -o yaml > deployment.yaml
kubectl create service clusterip my-svc --tcp=80:8080 --dry-run=client -o yaml > service.yaml
```

#### Deleting Resources

```bash
# Delete by name
kubectl delete pod my-pod
kubectl delete deployment my-deployment

# Delete from file
kubectl delete -f manifest.yaml

# Delete by label
kubectl delete pods -l app=nginx

# Delete all pods in a namespace
kubectl delete pods --all -n development

# Delete a namespace and everything in it
kubectl delete namespace development

# Force delete (use with caution)
kubectl delete pod my-pod --grace-period=0 --force
```

#### Logs and Debugging

```bash
# Pod logs
kubectl logs my-pod
kubectl logs my-pod -c my-container      # Specific container
kubectl logs -f my-pod                    # Follow (stream) logs
kubectl logs --tail=50 my-pod            # Last 50 lines
kubectl logs --since=1h my-pod           # Last hour
kubectl logs -l app=nginx --all-containers  # All pods with label
kubectl logs my-pod --previous           # Previous container instance

# Execute commands in a pod
kubectl exec my-pod -- ls /
kubectl exec -it my-pod -- /bin/bash     # Interactive shell
kubectl exec -it my-pod -c my-container -- /bin/sh  # Specific container

# Port forwarding
kubectl port-forward pod/my-pod 8080:80
kubectl port-forward svc/my-service 8080:80
kubectl port-forward deployment/my-deploy 8080:80

# Copy files
kubectl cp my-pod:/path/to/file ./local-file
kubectl cp ./local-file my-pod:/path/to/file
```

#### Scaling and Rolling Updates

```bash
# Scale
kubectl scale deployment my-deployment --replicas=5

# Autoscale
kubectl autoscale deployment my-deployment --min=3 --max=10 --cpu-percent=80

# Rollout management
kubectl rollout status deployment/my-deployment
kubectl rollout history deployment/my-deployment
kubectl rollout undo deployment/my-deployment
kubectl rollout restart deployment/my-deployment
kubectl rollout pause deployment/my-deployment
kubectl rollout resume deployment/my-deployment
```

#### Resource Usage

```bash
# Node resource usage (requires metrics-server)
kubectl top nodes

# Pod resource usage
kubectl top pods
kubectl top pods -n kube-system
kubectl top pods --sort-by=cpu
kubectl top pods --sort-by=memory
```

#### Context and Configuration

```bash
# View kubeconfig
kubectl config view

# View current context
kubectl config current-context

# List all contexts
kubectl config get-contexts

# Switch context
kubectl config use-context my-context

# Set default namespace for current context
kubectl config set-context --current --namespace=development

# Add a new cluster to kubeconfig
kubectl config set-cluster my-cluster --server=https://k8s.example.com
kubectl config set-credentials my-user --token=my-token
kubectl config set-context my-context --cluster=my-cluster --user=my-user
```

---

### YAML Manifests Best Practices

#### Labels and Selectors

Labels are key-value pairs attached to objects. They are used for organizing and selecting subsets of resources.

```yaml
metadata:
  labels:
    app: my-app                    # Application name
    version: v1.2.0                # Application version
    environment: production        # Deployment environment
    tier: frontend                 # Application tier
    team: platform                 # Owning team
    managed-by: helm               # Management tool
```

Recommended label conventions from Kubernetes:

```yaml
metadata:
  labels:
    app.kubernetes.io/name: my-app
    app.kubernetes.io/instance: my-app-prod
    app.kubernetes.io/version: "1.2.0"
    app.kubernetes.io/component: frontend
    app.kubernetes.io/part-of: my-platform
    app.kubernetes.io/managed-by: helm
```

#### Annotations

Annotations store non-identifying metadata. Unlike labels, they cannot be used for selection.

```yaml
metadata:
  annotations:
    kubernetes.io/change-cause: "Update to v1.2.0 with new login page"
    prometheus.io/scrape: "true"
    prometheus.io/port: "9090"
    description: "Frontend service for the platform"
```

#### Resource Requests and Limits

Always set resource requests and limits to ensure proper scheduling and prevent resource starvation.

```yaml
resources:
  requests:
    cpu: "100m"        # 100 millicores = 0.1 CPU core
    memory: "128Mi"    # 128 Mebibytes
  limits:
    cpu: "500m"        # 500 millicores = 0.5 CPU core
    memory: "256Mi"    # 256 Mebibytes
```

CPU units:
- `1` = 1 CPU core
- `500m` = 0.5 CPU core (500 millicores)
- `100m` = 0.1 CPU core (100 millicores)

Memory units:
- `128Mi` = 128 Mebibytes (1 Mi = 1,048,576 bytes)
- `1Gi` = 1 Gibibyte
- `128M` = 128 Megabytes (1 M = 1,000,000 bytes)

#### Liveness and Readiness Probes

**Liveness Probe**: Determines if a container is running. If it fails, the container is restarted.

**Readiness Probe**: Determines if a container is ready to accept traffic. If it fails, the pod is removed from Service endpoints.

**Startup Probe**: Used for slow-starting containers. Disables liveness/readiness checks until the startup probe succeeds.

```yaml
containers:
  - name: app
    image: my-app:v1
    ports:
      - containerPort: 8080

    livenessProbe:
      httpGet:
        path: /healthz
        port: 8080
      initialDelaySeconds: 15   # Wait before first check
      periodSeconds: 10         # Check every 10 seconds
      timeoutSeconds: 3         # Timeout for each check
      failureThreshold: 3       # Restart after 3 consecutive failures
      successThreshold: 1       # 1 success to be considered healthy

    readinessProbe:
      httpGet:
        path: /ready
        port: 8080
      initialDelaySeconds: 5
      periodSeconds: 5
      timeoutSeconds: 3
      failureThreshold: 3
      successThreshold: 1

    startupProbe:
      httpGet:
        path: /healthz
        port: 8080
      failureThreshold: 30     # 30 * 10 = 300 seconds max startup time
      periodSeconds: 10
```

Probe types:
- **httpGet**: HTTP request, success = 200-399 status code
- **tcpSocket**: TCP connection, success = port is open
- **exec**: Run a command, success = exit code 0
- **grpc**: gRPC health check (Kubernetes 1.27+)

#### Full Example with All Best Practices

```yaml
# best-practices-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
  namespace: production
  labels:
    app.kubernetes.io/name: my-app
    app.kubernetes.io/version: "1.2.0"
    app.kubernetes.io/component: backend
    app.kubernetes.io/managed-by: kubectl
  annotations:
    kubernetes.io/change-cause: "Deploy v1.2.0 with performance improvements"
spec:
  replicas: 3
  revisionHistoryLimit: 5
  selector:
    matchLabels:
      app.kubernetes.io/name: my-app
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
      maxSurge: 1
  template:
    metadata:
      labels:
        app.kubernetes.io/name: my-app
        app.kubernetes.io/version: "1.2.0"
        app.kubernetes.io/component: backend
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "9090"
    spec:
      serviceAccountName: my-app-sa
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
        fsGroup: 1000
      containers:
        - name: my-app
          image: myregistry.example.com/my-app:1.2.0   # Use specific tags, never :latest
          imagePullPolicy: IfNotPresent
          ports:
            - name: http
              containerPort: 8080
              protocol: TCP
            - name: metrics
              containerPort: 9090
              protocol: TCP
          env:
            - name: APP_ENV
              valueFrom:
                configMapKeyRef:
                  name: my-app-config
                  key: APP_ENV
            - name: DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: my-app-secrets
                  key: DB_PASSWORD
          resources:
            requests:
              cpu: "200m"
              memory: "256Mi"
            limits:
              cpu: "1000m"
              memory: "512Mi"
          livenessProbe:
            httpGet:
              path: /healthz
              port: http
            initialDelaySeconds: 15
            periodSeconds: 10
            timeoutSeconds: 3
            failureThreshold: 3
          readinessProbe:
            httpGet:
              path: /ready
              port: http
            initialDelaySeconds: 5
            periodSeconds: 5
            timeoutSeconds: 3
            failureThreshold: 3
          volumeMounts:
            - name: config-volume
              mountPath: /etc/config
              readOnly: true
          securityContext:
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: true
            capabilities:
              drop:
                - ALL
      volumes:
        - name: config-volume
          configMap:
            name: my-app-config
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
            - weight: 100
              podAffinityTerm:
                labelSelector:
                  matchExpressions:
                    - key: app.kubernetes.io/name
                      operator: In
                      values:
                        - my-app
                topologyKey: kubernetes.io/hostname
      terminationGracePeriodSeconds: 30
```

---

## 6. Deploying a Python Flask App

This section walks through deploying a Python Flask application step by step on Minikube.

### Prerequisites

- Minikube installed and running (`minikube start`)
- kubectl configured
- Ingress addon enabled (`minikube addons enable ingress`)

### The Flask Application

We will deploy a simple Flask app. Here is the assumed `Dockerfile` and application (from the Docker guide):

```python
# app.py
from flask import Flask
import os

app = Flask(__name__)

@app.route("/")
def hello():
    app_env = os.getenv("APP_ENV", "development")
    app_version = os.getenv("APP_VERSION", "1.0.0")
    return f"Hello from Flask! Environment: {app_env}, Version: {app_version}"

@app.route("/healthz")
def health():
    return "OK", 200

@app.route("/ready")
def ready():
    return "Ready", 200

if __name__ == "__main__":
    port = int(os.getenv("APP_PORT", 5000))
    app.run(host="0.0.0.0", port=port)
```

```dockerfile
# Dockerfile
FROM python:3.12-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
EXPOSE 5000
CMD ["python", "app.py"]
```

```
# requirements.txt
flask==3.1.*
```

### Step 1: Build and Load the Image

```bash
# Point Docker CLI to Minikube's Docker daemon
eval $(minikube docker-env)

# Build the image
docker build -t flask-app:v1.0.0 .

# Verify it is available
docker images | grep flask-app
```

### Step 2: Create a Namespace

```yaml
# k8s/namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: flask-app
  labels:
    app: flask-app
    environment: development
```

```bash
kubectl apply -f k8s/namespace.yaml
```

### Step 3: Create a ConfigMap

```yaml
# k8s/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: flask-app-config
  namespace: flask-app
data:
  APP_ENV: "development"
  APP_PORT: "5000"
  APP_VERSION: "1.0.0"
```

```bash
kubectl apply -f k8s/configmap.yaml
```

### Step 4: Create a Secret (Optional)

```yaml
# k8s/secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: flask-app-secret
  namespace: flask-app
type: Opaque
stringData:
  SECRET_KEY: "my-super-secret-key-change-in-production"
```

```bash
kubectl apply -f k8s/secret.yaml
```

### Step 5: Create a Deployment

```yaml
# k8s/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: flask-app
  namespace: flask-app
  labels:
    app: flask-app
  annotations:
    kubernetes.io/change-cause: "Initial deployment v1.0.0"
spec:
  replicas: 3
  selector:
    matchLabels:
      app: flask-app
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
      maxSurge: 1
  template:
    metadata:
      labels:
        app: flask-app
        version: v1.0.0
    spec:
      containers:
        - name: flask-app
          image: flask-app:v1.0.0
          imagePullPolicy: Never    # Use local image (important for Minikube)
          ports:
            - containerPort: 5000
              protocol: TCP
          envFrom:
            - configMapRef:
                name: flask-app-config
            - secretRef:
                name: flask-app-secret
          resources:
            requests:
              cpu: "100m"
              memory: "128Mi"
            limits:
              cpu: "250m"
              memory: "256Mi"
          livenessProbe:
            httpGet:
              path: /healthz
              port: 5000
            initialDelaySeconds: 10
            periodSeconds: 10
            failureThreshold: 3
          readinessProbe:
            httpGet:
              path: /ready
              port: 5000
            initialDelaySeconds: 5
            periodSeconds: 5
            failureThreshold: 3
```

```bash
kubectl apply -f k8s/deployment.yaml

# Verify the deployment
kubectl get deployment -n flask-app
kubectl get pods -n flask-app
kubectl describe deployment flask-app -n flask-app
```

### Step 6: Create a Service

```yaml
# k8s/service.yaml
apiVersion: v1
kind: Service
metadata:
  name: flask-app-service
  namespace: flask-app
  labels:
    app: flask-app
spec:
  type: ClusterIP
  selector:
    app: flask-app
  ports:
    - protocol: TCP
      port: 80
      targetPort: 5000
```

```bash
kubectl apply -f k8s/service.yaml

# Verify the service
kubectl get svc -n flask-app
kubectl describe svc flask-app-service -n flask-app
kubectl get endpoints flask-app-service -n flask-app
```

### Step 7: Test with Port-Forward

```bash
# Forward local port 8080 to the service
kubectl port-forward svc/flask-app-service 8080:80 -n flask-app

# In another terminal, test the app
curl http://localhost:8080
# Expected output: Hello from Flask! Environment: development, Version: 1.0.0

curl http://localhost:8080/healthz
# Expected output: OK

curl http://localhost:8080/ready
# Expected output: Ready
```

### Step 8: Expose with Ingress (on Minikube)

```yaml
# k8s/ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: flask-app-ingress
  namespace: flask-app
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
    - host: flask-app.local
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: flask-app-service
                port:
                  number: 80
```

```bash
# Apply the Ingress
kubectl apply -f k8s/ingress.yaml

# Get the Minikube IP
minikube ip

# Add the hostname to /etc/hosts
echo "$(minikube ip) flask-app.local" | sudo tee -a /etc/hosts

# Test the Ingress
curl http://flask-app.local
# Expected output: Hello from Flask! Environment: development, Version: 1.0.0

# On macOS with the Docker driver, you may need to use minikube tunnel instead:
minikube tunnel
# Then access http://flask-app.local in your browser
```

### Step 9: Verify Everything

```bash
# View all resources in the flask-app namespace
kubectl get all -n flask-app

# Expected output similar to:
# NAME                             READY   STATUS    RESTARTS   AGE
# pod/flask-app-6d5f8b7c4d-abc12   1/1     Running   0          5m
# pod/flask-app-6d5f8b7c4d-def34   1/1     Running   0          5m
# pod/flask-app-6d5f8b7c4d-ghi56   1/1     Running   0          5m
#
# NAME                        TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE
# service/flask-app-service   ClusterIP   10.96.45.123   <none>        80/TCP    5m
#
# NAME                        READY   UP-TO-DATE   AVAILABLE   AGE
# deployment.apps/flask-app   3/3     3            3           5m
#
# NAME                                   DESIRED   CURRENT   READY   AGE
# replicaset.apps/flask-app-6d5f8b7c4d   3         3         3       5m

# View pod logs
kubectl logs -l app=flask-app -n flask-app

# View ingress
kubectl get ingress -n flask-app
```

### Step 10: Perform a Rolling Update

```bash
# Build a new version
docker build -t flask-app:v1.1.0 .

# Update the deployment image
kubectl set image deployment/flask-app flask-app=flask-app:v1.1.0 -n flask-app

# Watch the rolling update
kubectl rollout status deployment/flask-app -n flask-app

# Verify the new pods
kubectl get pods -n flask-app

# If something goes wrong, rollback
kubectl rollout undo deployment/flask-app -n flask-app
```

### Complete Directory Structure

```
k8s/
  namespace.yaml
  configmap.yaml
  secret.yaml
  deployment.yaml
  service.yaml
  ingress.yaml
```

You can apply everything at once:

```bash
kubectl apply -f k8s/
```

---

## 7. Troubleshooting Kubernetes

### Pod in CrashLoopBackOff

The pod starts, crashes, Kubernetes restarts it, it crashes again, and the restart delay keeps increasing.

**Diagnosis:**

```bash
# Check pod status and restart count
kubectl get pods

# Check pod events and status details
kubectl describe pod <pod-name>

# Check container logs (current instance)
kubectl logs <pod-name>

# Check logs from the previous crashed instance
kubectl logs <pod-name> --previous

# Check if the container command/entrypoint is correct
kubectl get pod <pod-name> -o yaml | grep -A5 "command\|args"
```

**Common Causes:**
- Application error causing crash at startup
- Missing environment variables or configuration
- Failing liveness probe that restarts a healthy container
- Insufficient memory (OOMKilled) -- check `kubectl describe pod` for OOMKilled in "Last State"
- Wrong command or entrypoint in the container image

**Fixes:**
- Check and fix application code
- Verify all required env vars and ConfigMaps/Secrets exist
- Increase `initialDelaySeconds` on liveness probe
- Increase memory limits
- Verify the container image and command

---

### ImagePullBackOff

Kubernetes cannot pull the container image.

**Diagnosis:**

```bash
kubectl describe pod <pod-name>
# Look in the Events section for messages like:
# "Failed to pull image" or "repository does not exist"
```

**Common Causes:**
- Image name or tag is misspelled
- Image does not exist in the registry
- Private registry requires authentication (missing imagePullSecret)
- Network issues preventing access to the registry
- Using `imagePullPolicy: Always` with a local image on Minikube

**Fixes:**
```bash
# Verify the image exists
docker pull <image-name>:<tag>

# For private registries, create and reference an image pull secret
kubectl create secret docker-registry regcred \
  --docker-server=<registry-url> \
  --docker-username=<username> \
  --docker-password=<password>

# Reference it in your pod spec:
# spec:
#   imagePullSecrets:
#     - name: regcred

# For Minikube with local images, set:
# imagePullPolicy: Never
```

---

### Pending Pods

The pod is stuck in `Pending` state and never gets scheduled.

**Diagnosis:**

```bash
kubectl describe pod <pod-name>
# Check the Events section for scheduling failures

kubectl get nodes
kubectl describe node <node-name>
kubectl top nodes
```

**Common Causes:**
- Insufficient resources (CPU or memory) on any node
- Node taints that the pod does not tolerate
- NodeSelector or affinity rules that no node satisfies
- PersistentVolumeClaim is not bound (Pending PVC)
- ResourceQuota exceeded in the namespace

**Fixes:**
- Reduce resource requests on the pod
- Add more nodes or increase node capacity
- Add tolerations to the pod spec
- Fix nodeSelector or affinity rules
- Create or bind the required PersistentVolume
- Increase ResourceQuota limits

---

### Service Not Accessible

You created a Service but cannot reach your application.

**Diagnosis:**

```bash
# Verify the service exists and has endpoints
kubectl get svc <service-name>
kubectl get endpoints <service-name>

# If endpoints list is empty, the selector does not match any pods
kubectl get pods --show-labels
kubectl get svc <service-name> -o yaml | grep -A3 selector

# Test connectivity from within the cluster
kubectl run debug --rm -it --image=busybox:1.36 -- /bin/sh
# Inside the debug pod:
wget -qO- http://<service-name>.<namespace>.svc.cluster.local
nslookup <service-name>.<namespace>.svc.cluster.local

# For NodePort services
kubectl get svc <service-name>
# Access via <NodeIP>:<NodePort>

# For Minikube NodePort services
minikube service <service-name> --url
```

**Common Causes:**
- Service selector labels do not match pod labels
- Target port does not match the container port
- Pod is not in `Ready` state (readiness probe failing)
- NetworkPolicy blocking traffic
- Wrong namespace

---

### Common kubectl Debug Commands Cheat Sheet

```bash
# Quick health check of the cluster
kubectl get nodes
kubectl get pods -A | grep -v Running
kubectl get events --sort-by=.lastTimestamp -A | tail -20

# Check resource usage
kubectl top nodes
kubectl top pods -A --sort-by=cpu

# Run a temporary debug pod
kubectl run debug --rm -it --image=busybox:1.36 -- /bin/sh
kubectl run debug --rm -it --image=nicolaka/netshoot -- /bin/bash

# Debug a specific pod (ephemeral debug container, K8s 1.23+)
kubectl debug <pod-name> -it --image=busybox:1.36

# Check DNS resolution from inside the cluster
kubectl run dns-test --rm -it --image=busybox:1.36 -- nslookup kubernetes.default

# Check if a pod can reach a service
kubectl run curl-test --rm -it --image=curlimages/curl -- curl -s http://my-service.default.svc.cluster.local

# Dump all pod info as YAML for detailed inspection
kubectl get pod <pod-name> -o yaml

# Check container exit codes
kubectl get pod <pod-name> -o jsonpath='{.status.containerStatuses[*].lastState.terminated.exitCode}'
```

---

## 8. Practice Exercises

### Exercise 1: Deploy NGINX and Expose It

**Objective**: Deploy an NGINX web server, create a Service, and access it.

**Steps:**

1. Create a namespace called `exercise-1`:
```bash
kubectl create namespace exercise-1
```

2. Create a Deployment with 2 replicas of `nginx:1.27`:
```bash
kubectl create deployment nginx --image=nginx:1.27 --replicas=2 -n exercise-1
```

3. Verify the pods are running:
```bash
kubectl get pods -n exercise-1
```

4. Expose the deployment as a NodePort service on port 80:
```bash
kubectl expose deployment nginx --port=80 --type=NodePort -n exercise-1
```

5. Get the service URL and access it:
```bash
# On Minikube:
minikube service nginx -n exercise-1 --url
# Open the URL in your browser or use curl

# Or use port-forward:
kubectl port-forward svc/nginx 8080:80 -n exercise-1
curl http://localhost:8080
```

6. Scale the deployment to 5 replicas:
```bash
kubectl scale deployment nginx --replicas=5 -n exercise-1
kubectl get pods -n exercise-1
```

7. Clean up:
```bash
kubectl delete namespace exercise-1
```

---

### Exercise 2: ConfigMaps and Secrets in Practice

**Objective**: Create a Pod that reads its configuration from a ConfigMap and a Secret.

**Steps:**

1. Create a namespace:
```bash
kubectl create namespace exercise-2
```

2. Create a ConfigMap with application settings:
```bash
kubectl create configmap app-settings \
  --from-literal=APP_COLOR=blue \
  --from-literal=APP_MODE=production \
  -n exercise-2
```

3. Create a Secret with a database password:
```bash
kubectl create secret generic db-credentials \
  --from-literal=DB_USER=admin \
  --from-literal=DB_PASS=supersecret123 \
  -n exercise-2
```

4. Create a pod that uses both (save as `exercise-2-pod.yaml`):
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: config-test-pod
  namespace: exercise-2
spec:
  containers:
    - name: test
      image: busybox:1.36
      command: ["sh", "-c", "echo APP_COLOR=$APP_COLOR APP_MODE=$APP_MODE DB_USER=$DB_USER DB_PASS=$DB_PASS && sleep 3600"]
      envFrom:
        - configMapRef:
            name: app-settings
        - secretRef:
            name: db-credentials
```

```bash
kubectl apply -f exercise-2-pod.yaml
```

5. Verify the environment variables are set:
```bash
kubectl logs config-test-pod -n exercise-2
# Expected: APP_COLOR=blue APP_MODE=production DB_USER=admin DB_PASS=supersecret123

kubectl exec config-test-pod -n exercise-2 -- env | grep -E "APP_|DB_"
```

6. Clean up:
```bash
kubectl delete namespace exercise-2
```

---

### Exercise 3: Rolling Updates and Rollbacks

**Objective**: Deploy an application, perform a rolling update, and practice rollback.

**Steps:**

1. Create a namespace:
```bash
kubectl create namespace exercise-3
```

2. Deploy nginx version 1.25:
```bash
kubectl create deployment web --image=nginx:1.25 --replicas=3 -n exercise-3
kubectl annotate deployment web kubernetes.io/change-cause="Deploy nginx 1.25" -n exercise-3
```

3. Verify the deployment:
```bash
kubectl get deployment web -n exercise-3
kubectl get pods -n exercise-3
```

4. Perform a rolling update to nginx 1.26:
```bash
kubectl set image deployment/web nginx=nginx:1.26 -n exercise-3
kubectl annotate deployment web kubernetes.io/change-cause="Update to nginx 1.26" --overwrite -n exercise-3
kubectl rollout status deployment/web -n exercise-3
```

5. Update again to nginx 1.27:
```bash
kubectl set image deployment/web nginx=nginx:1.27 -n exercise-3
kubectl annotate deployment web kubernetes.io/change-cause="Update to nginx 1.27" --overwrite -n exercise-3
kubectl rollout status deployment/web -n exercise-3
```

6. View the rollout history:
```bash
kubectl rollout history deployment/web -n exercise-3
```

7. Roll back to nginx 1.26 (revision 2):
```bash
kubectl rollout undo deployment/web --to-revision=2 -n exercise-3
kubectl rollout status deployment/web -n exercise-3
```

8. Verify the rollback:
```bash
kubectl describe deployment web -n exercise-3 | grep Image
# Should show nginx:1.26
```

9. Clean up:
```bash
kubectl delete namespace exercise-3
```

---

### Exercise 4: Persistent Storage

**Objective**: Deploy a Pod with persistent storage that survives pod deletion.

**Steps:**

1. Create a namespace:
```bash
kubectl create namespace exercise-4
```

2. Create a PersistentVolumeClaim (save as `exercise-4-pvc.yaml`):
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: data-pvc
  namespace: exercise-4
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
```

```bash
kubectl apply -f exercise-4-pvc.yaml
kubectl get pvc -n exercise-4
```

3. Create a Pod that writes data to the volume (save as `exercise-4-writer.yaml`):
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: writer-pod
  namespace: exercise-4
spec:
  containers:
    - name: writer
      image: busybox:1.36
      command: ["sh", "-c", "echo 'Data written at '$(date) > /data/message.txt && cat /data/message.txt && sleep 3600"]
      volumeMounts:
        - name: data-volume
          mountPath: /data
  volumes:
    - name: data-volume
      persistentVolumeClaim:
        claimName: data-pvc
```

```bash
kubectl apply -f exercise-4-writer.yaml
kubectl logs writer-pod -n exercise-4
```

4. Delete the writer pod:
```bash
kubectl delete pod writer-pod -n exercise-4
```

5. Create a new Pod that reads the data (save as `exercise-4-reader.yaml`):
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: reader-pod
  namespace: exercise-4
spec:
  containers:
    - name: reader
      image: busybox:1.36
      command: ["sh", "-c", "cat /data/message.txt && sleep 3600"]
      volumeMounts:
        - name: data-volume
          mountPath: /data
  volumes:
    - name: data-volume
      persistentVolumeClaim:
        claimName: data-pvc
```

```bash
kubectl apply -f exercise-4-reader.yaml
kubectl logs reader-pod -n exercise-4
# The data from the deleted writer-pod should still be there
```

6. Clean up:
```bash
kubectl delete namespace exercise-4
```

---

### Exercise 5: Multi-Service Application with Ingress

**Objective**: Deploy two services (frontend and API) and route traffic using Ingress.

**Steps:**

1. Create a namespace:
```bash
kubectl create namespace exercise-5
```

2. Enable the Ingress addon on Minikube:
```bash
minikube addons enable ingress
```

3. Deploy a frontend service (save as `exercise-5-frontend.yaml`):
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
  namespace: exercise-5
spec:
  replicas: 2
  selector:
    matchLabels:
      app: frontend
  template:
    metadata:
      labels:
        app: frontend
    spec:
      containers:
        - name: nginx
          image: nginx:1.27
          ports:
            - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: frontend-service
  namespace: exercise-5
spec:
  type: ClusterIP
  selector:
    app: frontend
  ports:
    - port: 80
      targetPort: 80
```

4. Deploy an API service (save as `exercise-5-api.yaml`):
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api
  namespace: exercise-5
spec:
  replicas: 2
  selector:
    matchLabels:
      app: api
  template:
    metadata:
      labels:
        app: api
    spec:
      containers:
        - name: api
          image: hashicorp/http-echo
          args:
            - "-text=Hello from the API service"
          ports:
            - containerPort: 5678
---
apiVersion: v1
kind: Service
metadata:
  name: api-service
  namespace: exercise-5
spec:
  type: ClusterIP
  selector:
    app: api
  ports:
    - port: 80
      targetPort: 5678
```

5. Create an Ingress with path-based routing (save as `exercise-5-ingress.yaml`):
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app-ingress
  namespace: exercise-5
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
    - host: exercise5.local
      http:
        paths:
          - path: /api
            pathType: Prefix
            backend:
              service:
                name: api-service
                port:
                  number: 80
          - path: /
            pathType: Prefix
            backend:
              service:
                name: frontend-service
                port:
                  number: 80
```

6. Apply everything:
```bash
kubectl apply -f exercise-5-frontend.yaml
kubectl apply -f exercise-5-api.yaml
kubectl apply -f exercise-5-ingress.yaml
```

7. Add the hostname to `/etc/hosts`:
```bash
echo "$(minikube ip) exercise5.local" | sudo tee -a /etc/hosts
```

8. Test the routing:
```bash
# Frontend (should return NGINX default page)
curl http://exercise5.local/

# API (should return "Hello from the API service")
curl http://exercise5.local/api
```

9. Verify all resources:
```bash
kubectl get all,ingress -n exercise-5
```

10. Clean up:
```bash
kubectl delete namespace exercise-5
# Remove the /etc/hosts entry manually
```

---

## Next Steps

After mastering the fundamentals covered in this guide, consider exploring:

- **Helm** -- Kubernetes package manager for templating and managing complex deployments
- **StatefulSets** -- For stateful applications like databases
- **DaemonSets** -- Run a pod on every node (log collectors, monitoring agents)
- **Jobs and CronJobs** -- For batch and scheduled workloads
- **RBAC** -- Role-Based Access Control for securing your cluster
- **NetworkPolicies** -- Fine-grained network access control between pods
- **Horizontal Pod Autoscaler (HPA)** -- Automatically scale based on metrics
- **Service Mesh (Istio, Linkerd)** -- Advanced traffic management, observability, security
- **GitOps (ArgoCD, Flux)** -- Declarative, Git-driven deployment workflows
- **Operators** -- Extend Kubernetes with custom controllers for application-specific logic
