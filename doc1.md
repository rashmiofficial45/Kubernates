# 🚀 Kubernetes (k8s) — Beginner's Guide

> **Pre-requisite:** Make sure you have **Docker** installed before starting Kubernetes.

---

## 📖 What is Kubernetes?

Kubernetes (also called **k8s** — "K" + 8 letters + "s") is a **container orchestration engine** that lets you:

- Deploy Docker images in a cloud-native way
- Auto-heal crashed containers
- Auto-scale your applications
- Monitor your system via a dashboard

---

## 🧠 Key Concepts (Jargon)

| Term | What it means |
|------|--------------|
| **Cluster** | A group of machines (nodes) running Kubernetes together |
| **Node** | A single machine in a cluster |
| **Master Node** | The brain — manages deployments, healing, scheduling |
| **Worker Node** | Where your actual containers run |
| **Image** | A Docker image (e.g., `nginx`, `postgres`) |
| **Container** | A running instance of an image |
| **Pod** | The smallest unit in k8s — wraps one or more containers |
| **ReplicaSet** | Ensures a set number of identical pods are always running |
| **Deployment** | Manages ReplicaSets — handles updates, rollbacks, scaling |
| **Service** | Exposes pods to the network (internally or externally) |

### Master Node Components

```
Master Node
├── API Server          → Handles all requests (from kubectl or other tools)
├── etcd                → Key-value store — stores all cluster state
├── kube-scheduler      → Decides which node a new pod runs on
└── kube-controller-manager
    ├── Deployment Controller
    ├── ReplicaSet Controller
    └── Node Controller
```

### Worker Node Components

```
Worker Node
├── kubelet             → Agent that ensures containers are running in pods
├── kube-proxy          → Handles network routing to pods
└── Container Runtime   → Runs containers (containerd, CRI-O, or Docker)
```

---

## ⚙️ Step 1 — Install Tools

### Install `kind` (Kubernetes in Docker)

Follow the official guide: https://kind.sigs.k8s.io/docs/user/quick-start/#installation

### Install `kubectl` (Kubernetes CLI)

Follow the official guide: https://kubernetes.io/docs/tasks/tools/#kubectl

---

## 🏗️ Step 2 — Create a Local Cluster

### Option A: Single Node (Simple)

```bash
kind create cluster --name local
```

Check running Docker containers (you'll see the control-plane):

```bash
docker ps
```

Delete the cluster when done:

```bash
kind delete cluster -n local
```

---

### Option B: Multi-Node Cluster (Recommended)

Create a file called `clusters.yml`:

```yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
  - role: control-plane
  - role: worker
  - role: worker
```

Create the cluster:

```bash
kind create cluster --config clusters.yml --name local
```

Verify all 3 containers are running:

```bash
docker ps
```

---

## 🔍 Step 3 — Verify kubectl Works

```bash
kubectl get nodes     # List all nodes in the cluster
kubectl get pods      # List all running pods
```

---

## 🐋 Step 4 — Create Your First Pod

A **Pod** is the smallest deployable unit — it wraps your container.

### Method 1: Quick Command

```bash
kubectl run nginx --image=nginx --port=80
```

### Method 2: Using a Manifest File (Recommended)

Create `manifest.yml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  containers:
    - name: nginx
      image: nginx
      ports:
        - containerPort: 80
```

Apply it:

```bash
kubectl apply -f manifest.yml
```

### Useful Pod Commands

```bash
kubectl get pods                    # See all pods and their status
kubectl logs nginx                  # View logs of the pod
kubectl describe pod nginx          # Detailed info (which node, IP, events)
kubectl delete pod nginx            # Delete the pod
```

---

## 🔁 Step 5 — Create a ReplicaSet

A **ReplicaSet** ensures a fixed number of pods are always running. If one dies, it spins up a new one automatically.

Create `rs.yml`:

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: nginx-replicaset
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
          image: nginx:latest
          ports:
            - containerPort: 80
```

```bash
kubectl apply -f rs.yml
kubectl get rs                      # View ReplicaSet status
kubectl get pods                    # See 3 pods running
```

> **Try it:** Delete a pod — the ReplicaSet will automatically create a new one!

```bash
kubectl delete pod <pod-name>
kubectl get pods                    # A new pod appears automatically ✅
```

Clean up:

```bash
kubectl delete rs nginx-replicaset
```

---

## 🚀 Step 6 — Create a Deployment

A **Deployment** is a higher-level abstraction that manages ReplicaSets. It adds:
- ✅ Rolling updates
- ✅ Rollbacks
- ✅ Zero-downtime deployments

Create `deployment.yml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
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
          image: nginx:latest
          ports:
            - containerPort: 80
```

```bash
kubectl apply -f deployment.yml
kubectl get deployment              # View deployment
kubectl get rs                      # Deployment created a ReplicaSet
kubectl get pods                    # ReplicaSet created 3 pods
```

### Useful Deployment Commands

```bash
kubectl rollout history deployment/nginx-deployment    # View deployment history
kubectl rollout undo deployment/nginx-deployment       # Rollback to previous version
```

### Why Deployment > ReplicaSet?

| Feature | ReplicaSet | Deployment |
|---------|-----------|------------|
| Keeps pods running | ✅ | ✅ |
| Rolling updates | ❌ | ✅ |
| Rollback on failure | ❌ | ✅ |
| Manages ReplicaSets | ❌ | ✅ |

> **Example:** If you update the image to a bad one, Deployment keeps the old pods running and only stops when the new ones are healthy.

### Adding Environment Variables

```yaml
containers:
  - name: postgres
    image: postgres:latest
    ports:
      - containerPort: 5432
    env:
      - name: POSTGRES_PASSWORD
        value: "yourpassword"
```

---

## 🌐 Step 7 — Expose Your App with a Service

Pods have private IPs — you can't access them from outside the cluster directly. A **Service** solves this.

### Service Types

| Type | Description |
|------|-------------|
| **ClusterIP** | Internal only — pods talk to each other |
| **NodePort** | Exposes app on a port of each node (good for local dev) |
| **LoadBalancer** | Provisions a cloud load balancer (AWS, GCP, Azure) |

---

### NodePort Service (Local Dev)

First, recreate your cluster with a port exposed. Create `kind.yml`:

```yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
  - role: control-plane
    extraPortMappings:
      - containerPort: 30007
        hostPort: 30007
  - role: worker
  - role: worker
```

```bash
kind create cluster --config kind.yml --name local
kubectl apply -f deployment.yml
```

Create `service.yml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  selector:
    app: nginx
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
      nodePort: 30007
  type: NodePort
```

```bash
kubectl apply -f service.yml
```

Visit: **http://localhost:30007** 🎉

---

### LoadBalancer Service (Cloud / Production)

Create `service-lb.yml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  selector:
    app: nginx
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
  type: LoadBalancer
```

```bash
kubectl apply -f service-lb.yml
```

On a cloud provider (GKE, AWS EKS, Vultr), this automatically provisions a public load balancer.

---

## 🗺️ Full Flow Summary

```
Developer
   │
   ▼ kubectl apply
API Server (Master Node)
   │
   ├──▶ etcd (stores desired state)
   │
   ├──▶ Deployment Controller
   │         │
   │         ▼
   │     ReplicaSet Controller
   │         │
   │         ▼
   │     kube-scheduler (picks which node)
   │         │
   ▼         ▼
Worker Nodes
   ├── kubelet (starts containers via Container Runtime)
   └── kube-proxy (handles network routing)
         │
         ▼
       Pods (your running containers)
```

---

## 📋 Quick Reference Cheatsheet

```bash
# Cluster
kind create cluster --name local                          # Create cluster
kind delete cluster -n local                              # Delete cluster

# Pods
kubectl run nginx --image=nginx --port=80                 # Create a pod
kubectl get pods                                          # List pods
kubectl get pods -o wide                                  # List pods with IPs & nodes
kubectl describe pod <name>                               # Pod details
kubectl logs <pod-name>                                   # View logs
kubectl delete pod <name>                                 # Delete pod

# ReplicaSets
kubectl get rs                                            # List replicasets
kubectl delete rs <name>                                  # Delete replicaset

# Deployments
kubectl apply -f deployment.yml                           # Apply/update deployment
kubectl get deployment                                    # List deployments
kubectl rollout history deployment/<name>                 # Deployment history
kubectl rollout undo deployment/<name>                    # Rollback deployment
kubectl delete deployment <name>                          # Delete deployment

# Services
kubectl apply -f service.yml                              # Apply service
kubectl get services                                      # List services

# Debugging
kubectl get nodes                                         # List nodes
kubectl get all                                           # List everything
kubectl get pods --v=8                                    # Verbose output (HTTP requests)
```

---

## 🔗 Useful Links

- Kubernetes Docs: https://kubernetes.io/docs/concepts/overview/components/
- kind Quick Start: https://kind.sigs.k8s.io/docs/user/quick-start/
- minikube: https://minikube.sigs.k8s.io/docs/start/
- kubectl Install: https://kubernetes.io/docs/tasks/tools/#kubectl
- Docker Hub (Images): https://hub.docker.com