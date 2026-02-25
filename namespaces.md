# 🗂️ Kubernetes Namespaces — Complete Guide

> **Namespaces** are Kubernetes's way of creating virtual sub-clusters within a single physical cluster. Think of it as folders on your computer — same machine, but isolated spaces.

---

## 📌 What is a Namespace?

A **Namespace** is a mechanism for isolating groups of resources within a single Kubernetes cluster.

- Resources within the same namespace must have **unique names**.
- Resources in **different namespaces** can have the same name without conflict.
- Namespaces provide a scope for **names, RBAC policies, resource quotas**, and **network policies**.

```
┌──────────────────────────────────────────────────────────┐
│                   Kubernetes Cluster                     │
│                                                          │
│  ┌──────────────┐  ┌──────────────┐  ┌───────────────┐   │
│  │  namespace:  │  │  namespace:  │  │  namespace:   │   │
│  │    dev       │  │   staging    │  │  production   │   │
│  │              │  │              │  │               │   │
│  │  Pod: app-v1 │  │  Pod: app-v2 │  │  Pod: app-v3  │   │
│  │  Pod: db     │  │  Pod: db     │  │  Pod: db      │   │
│  │  Svc: api    │  │  Svc: api    │  │  Svc: api     │   │
│  └──────────────┘  └──────────────┘  └───────────────┘   │
└──────────────────────────────────────────────────────────┘
```

---

## 💡 Why Do We Need Namespaces?

### Problem Without Namespaces

In a large organization with multiple teams and environments, all resources dumped into one flat space leads to:

- **Naming conflicts** — Two teams both want a resource named `backend`
- **No isolation** — Team A can accidentally delete Team B's resources
- **No resource limits** — One greedy app can consume all cluster resources
- **No access control** — Every developer sees and modifies everything

### Solution With Namespaces

| Use Case | Without Namespace | With Namespace |
|---|---|---|
| Multiple teams | Name collisions | Each team gets their own namespace |
| Environments | Separate clusters needed | `dev`, `staging`, `prod` namespaces |
| Resource limits | Global or none | Per-namespace quotas |
| Access control | Cluster-wide | Namespace-scoped RBAC |
| Cost tracking | Impossible | Namespace-level billing |

---

## 🏗️ Default Namespaces in Kubernetes

When you spin up a Kubernetes cluster, **4 namespaces are created automatically**:

### 1. `default`
- The namespace used when you don't specify one
- Where your resources go if you don't define `namespace:` in your YAML

### 2. `kube-system`
- Reserved for **Kubernetes internal components**
- Contains: `kube-dns`, `kube-proxy`, `coredns`, controller-manager, scheduler, etcd
- ⚠️ **Never deploy your apps here**

### 3. `kube-public`
- Readable by **all users** (including unauthenticated ones)
- Used for cluster information that should be publicly accessible
- Contains `cluster-info` ConfigMap

### 4. `kube-node-lease`
- Contains **Lease objects** for each node
- Used for node heartbeat detection — helps the control plane detect node failures faster

```bash
# See all default namespaces
kubectl get namespaces

# Output:
# NAME              STATUS   AGE
# default           Active   5d
# kube-node-lease   Active   5d
# kube-public       Active   5d
# kube-system       Active   5d
```

---

## 🔍 What's Inside `kube-system`?

```bash
kubectl get pods -n kube-system

# You'll see Kubernetes internals like:
# coredns-*              → DNS resolution inside the cluster
# etcd-*                 → The cluster's key-value state store
# kube-apiserver-*       → The API server (front-end for control plane)
# kube-controller-manager-* → Runs all the controllers
# kube-scheduler-*       → Decides which node a pod goes to
# kube-proxy-*           → Handles networking on each node
```

---

## 🛠️ Namespace Commands (kubectl)

### Creating Namespaces

```bash
# Imperatively (quick)
kubectl create namespace dev
kubectl create namespace staging
kubectl create namespace production

# Declaratively (recommended for production)
kubectl apply -f namespace.yaml
```

**Declarative YAML:**

```yaml
# namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: dev
  labels:
    environment: development
    team: backend
```

---

### Listing & Inspecting Namespaces

```bash
# List all namespaces
kubectl get namespaces
kubectl get ns                            # Short alias

# Get details about a specific namespace
kubectl describe namespace dev

# Get namespace in YAML format
kubectl get namespace dev -o yaml

# Check namespace status (Active/Terminating)
kubectl get ns dev -o jsonpath='{.status.phase}'
```

---

### Working Within a Namespace

```bash
# Get pods in a specific namespace
kubectl get pods -n dev
kubectl get pods --namespace=staging

# Get ALL resources in a namespace
kubectl get all -n dev

# Get resources across ALL namespaces
kubectl get pods -A
kubectl get pods --all-namespaces

# Create a pod in a specific namespace
kubectl run nginx --image=nginx -n dev

# Apply a manifest to a specific namespace
kubectl apply -f deployment.yaml -n staging
```

---

### Setting a Default Namespace (Context)

Tired of typing `-n dev` every time? Set a sticky namespace:

```bash
# Set default namespace for current context
kubectl config set-context --current --namespace=dev

# Verify what namespace you're in
kubectl config view --minify | grep namespace

# Now all commands default to 'dev':
kubectl get pods           # equivalent to: kubectl get pods -n dev
kubectl get services       # equivalent to: kubectl get services -n dev
```

---

### Switching Namespaces with `kubens` (Recommended Tool)

```bash
# Install kubectx/kubens
brew install kubectx    # macOS

# List namespaces and switch interactively
kubens

# Switch to a specific namespace
kubens dev
kubens production

# Switch back to previous namespace
kubens -
```

---

### Deleting Namespaces

```bash
# Delete a namespace (WARNING: deletes ALL resources inside it!)
kubectl delete namespace dev

# Delete using a manifest
kubectl delete -f namespace.yaml

# Check if namespace is stuck in "Terminating"
kubectl get ns dev
```

> ⚠️ **Warning:** Deleting a namespace is irreversible and cascades — all Pods, Deployments, Services, ConfigMaps, Secrets inside it are deleted permanently.

---

## 📝 Specifying Namespace in YAML Manifests

Always define the namespace inside your resource manifests for clarity and GitOps compatibility:

```yaml
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend-api
  namespace: production        # ← Namespace declared here
  labels:
    app: backend
    env: production
spec:
  replicas: 3
  selector:
    matchLabels:
      app: backend
  template:
    metadata:
      labels:
        app: backend
    spec:
      containers:
        - name: api
          image: myapp/backend:v2.0
          ports:
            - containerPort: 8080
```

```yaml
# service.yaml
apiVersion: v1
kind: Service
metadata:
  name: backend-service
  namespace: production        # ← Same namespace as the deployment
spec:
  selector:
    app: backend
  ports:
    - port: 80
      targetPort: 8080
  type: ClusterIP
```

---

## 🌐 Cross-Namespace Communication

Services within the **same namespace** talk to each other using just the service name:

```
http://backend-service          # Within same namespace
```

Services in **different namespaces** must use the full DNS name:

```
http://<service-name>.<namespace>.svc.cluster.local
```

### Real Example

```
# Frontend in 'dev' namespace calling backend in 'dev' namespace:
http://backend-service

# Frontend in 'staging' namespace calling backend in 'production' namespace:
http://backend-service.production.svc.cluster.local

# Full DNS format breakdown:
# backend-service     → Service name
# production          → Namespace name
# svc                 → Resource type (service)
# cluster.local       → Cluster domain
```

```yaml
# Example: frontend deployment calling cross-namespace backend
env:
  - name: BACKEND_URL
    value: "http://backend-service.production.svc.cluster.local:8080"
```

---

## 🔒 Resource Quotas — Limiting Namespace Resources

Prevent one team or app from consuming all cluster resources:

```yaml
# resource-quota.yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: dev-quota
  namespace: dev
spec:
  hard:
    pods: "20"                  # Max 20 pods
    requests.cpu: "4"           # Max 4 CPU cores total (requests)
    requests.memory: 8Gi        # Max 8Gi memory total (requests)
    limits.cpu: "8"             # Max 8 CPU cores total (limits)
    limits.memory: 16Gi         # Max 16Gi memory total (limits)
    services: "10"              # Max 10 services
    persistentvolumeclaims: "5" # Max 5 PVCs
    services.loadbalancers: "2" # Max 2 LoadBalancer services
```

```bash
# Apply the quota
kubectl apply -f resource-quota.yaml

# View quota usage
kubectl describe quota dev-quota -n dev
kubectl get resourcequota -n dev

# Output shows current usage vs limits:
# Name:                   dev-quota
# Namespace:              dev
# Resource                Used    Hard
# --------                ----    ----
# limits.cpu              500m    8
# limits.memory           256Mi   16Gi
# pods                    2       20
# requests.cpu            250m    4
# requests.memory         128Mi   8Gi
```

---

## 📏 LimitRange — Default Resource Limits Per Container

Set default CPU/memory limits for containers in a namespace so developers don't forget:

```yaml
# limit-range.yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: dev-limit-range
  namespace: dev
spec:
  limits:
    - type: Container
      default:
        cpu: "500m"
        memory: "256Mi"
      defaultRequest:
        cpu: "100m"
        memory: "128Mi"
      max:
        cpu: "2"
        memory: "1Gi"
      min:
        cpu: "50m"
        memory: "64Mi"
```

```bash
kubectl apply -f limit-range.yaml
kubectl describe limitrange dev-limit-range -n dev
```

---

## 🔐 RBAC with Namespaces — Access Control

Namespaces enable fine-grained access control — give a developer access **only** to the `dev` namespace:

```yaml
# role.yaml — What actions are allowed
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: developer-role
  namespace: dev                # Scoped to 'dev' namespace only
rules:
  - apiGroups: ["", "apps"]
    resources: ["pods", "deployments", "services", "logs"]
    verbs: ["get", "list", "watch", "create", "update", "patch"]
  - apiGroups: [""]
    resources: ["pods/log"]
    verbs: ["get", "list"]

---
# rolebinding.yaml — Who gets the role
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: developer-binding
  namespace: dev
subjects:
  - kind: User
    name: alice                 # Developer's username
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: developer-role
  apiGroup: rbac.authorization.k8s.io
```

```bash
kubectl apply -f role.yaml
kubectl apply -f rolebinding.yaml

# Verify — can alice get pods in dev?
kubectl auth can-i get pods -n dev --as=alice      # yes
kubectl auth can-i get pods -n production --as=alice  # no ← isolated!

# List all roles in a namespace
kubectl get roles -n dev
kubectl get rolebindings -n dev
```

---

## 🏢 Real-World Namespace Strategies

### Strategy 1: By Environment
```
namespaces/
├── development
├── staging
└── production
```

### Strategy 2: By Team
```
namespaces/
├── team-backend
├── team-frontend
├── team-data
└── team-devops
```

### Strategy 3: By App + Environment (Recommended for large orgs)
```
namespaces/
├── payments-dev
├── payments-staging
├── payments-prod
├── auth-dev
├── auth-prod
└── analytics-prod
```

---

## 🚫 What Namespaces DON'T Isolate

Some resources are **cluster-scoped** and are NOT isolated by namespaces:

| Cluster-Scoped Resources | Why |
|---|---|
| `Nodes` | Physical/virtual machines — shared across all namespaces |
| `PersistentVolumes` | Storage volumes — cluster-wide |
| `ClusterRole` | Cluster-wide roles |
| `ClusterRoleBinding` | Cluster-wide role bindings |
| `StorageClass` | Storage class definitions |
| `IngressClass` | Ingress class definitions |
| `Namespace` itself | You can't namespace a namespace |

```bash
# Check if a resource is namespace-scoped or cluster-scoped
kubectl api-resources --namespaced=true    # namespace-scoped
kubectl api-resources --namespaced=false   # cluster-scoped
```

---

## ⚙️ Full Workflow Example — Multi-Team Setup

```bash
# Step 1: Create namespaces for two teams
kubectl create namespace team-alpha
kubectl create namespace team-beta

# Step 2: Set resource quotas per team
kubectl apply -f - <<EOF
apiVersion: v1
kind: ResourceQuota
metadata:
  name: alpha-quota
  namespace: team-alpha
spec:
  hard:
    pods: "10"
    requests.cpu: "2"
    requests.memory: 4Gi
EOF

# Step 3: Deploy apps to their respective namespaces
kubectl create deployment frontend --image=nginx -n team-alpha
kubectl create deployment backend --image=node:18 -n team-beta

# Step 4: Expose services within each namespace
kubectl expose deployment frontend --port=80 -n team-alpha
kubectl expose deployment backend --port=3000 -n team-beta

# Step 5: Verify isolation — each team sees only their resources
kubectl get all -n team-alpha
kubectl get all -n team-beta

# Step 6: Cross-namespace call from team-alpha to team-beta's backend
# URL: http://backend.team-beta.svc.cluster.local:3000
```

---

## 🔗 Useful Resources

- [Official Kubernetes Namespace Docs](https://kubernetes.io/docs/concepts/overview/working-with-objects/namespaces/)
- [Resource Quotas](https://kubernetes.io/docs/concepts/policy/resource-quotas/)
- [LimitRange](https://kubernetes.io/docs/concepts/policy/limit-range/)
- [kubens — Easy namespace switching](https://github.com/ahmetb/kubectx)

---

*Part of the Kubernetes Learning Series ☸️*
