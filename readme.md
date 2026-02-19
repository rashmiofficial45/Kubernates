# ☸️ Kubernetes (K8s) — Complete Cheat Sheet & Reference Guide

> A comprehensive reference for Kubernetes concepts, commands, jargon, and best practices.

---

## 📚 Table of Contents

1. [What is Kubernetes?](#what-is-kubernetes)
2. [Core Concepts & Jargon](#core-concepts--jargon)
3. [Architecture Overview](#architecture-overview)
4. [kubectl — CLI Reference](#kubectl--cli-reference)
   - [Cluster Info](#cluster-info)
   - [Namespaces](#namespaces)
   - [Pods](#pods)
   - [Deployments](#deployments)
   - [Services](#services)
   - [ConfigMaps & Secrets](#configmaps--secrets)
   - [Persistent Volumes](#persistent-volumes)
   - [Ingress](#ingress)
   - [Logs & Debugging](#logs--debugging)
   - [Scaling & Rollouts](#scaling--rollouts)
   - [Resource Management](#resource-management)
5. [YAML Manifests — Templates](#yaml-manifests--templates)
   - [Pod](#pod-manifest)
   - [Deployment](#deployment-manifest)
   - [Service](#service-manifest)
   - [ConfigMap](#configmap-manifest)
   - [Secret](#secret-manifest)
   - [Ingress](#ingress-manifest)
   - [PersistentVolumeClaim](#persistentvolumeclaim-manifest)
   - [HorizontalPodAutoscaler](#horizontalpodautoscaler-manifest)
6. [Namespaces & Context Management](#namespaces--context-management)
7. [RBAC — Role-Based Access Control](#rbac--role-based-access-control)
8. [Networking Concepts](#networking-concepts)
9. [Storage Concepts](#storage-concepts)
10. [Helm — Package Manager](#helm--package-manager)
11. [Common Patterns & Tips](#common-patterns--tips)
12. [Glossary](#glossary)

---

## What is Kubernetes?

Kubernetes (K8s) is an open-source **container orchestration platform** originally developed by Google. It automates the deployment, scaling, and management of containerized applications across a cluster of machines.

**Key capabilities:**
- Self-healing (restarts failed containers)
- Horizontal scaling
- Automated rollouts and rollbacks
- Service discovery and load balancing
- Secret and configuration management

---

## Core Concepts & Jargon

| Term | Description |
|------|-------------|
| **Cluster** | A set of machines (nodes) running Kubernetes. Consists of a Control Plane and worker nodes. |
| **Node** | A physical or virtual machine in the cluster that runs workloads (Pods). |
| **Pod** | The smallest deployable unit in K8s. A Pod wraps one or more containers that share network and storage. |
| **Container** | A lightweight, isolated runtime environment (e.g., Docker). Pods run containers. |
| **Deployment** | Manages a ReplicaSet and ensures the desired number of Pod replicas are always running. |
| **ReplicaSet** | Ensures a specified number of Pod replicas are running at any given time. Usually managed by a Deployment. |
| **StatefulSet** | Like a Deployment but for stateful apps — provides stable network IDs and persistent storage per Pod. |
| **DaemonSet** | Ensures a copy of a Pod runs on all (or selected) nodes. Used for logging, monitoring agents, etc. |
| **Job** | Runs a Pod to completion (one-time task). |
| **CronJob** | Runs a Job on a scheduled (cron) basis. |
| **Service** | An abstraction that exposes a set of Pods as a network service with a stable IP/DNS. |
| **Ingress** | Manages external HTTP/HTTPS access to services in the cluster, usually with routing rules. |
| **Namespace** | A virtual cluster within a cluster. Used to isolate resources between teams/environments. |
| **ConfigMap** | Stores non-sensitive configuration data as key-value pairs, injected into Pods. |
| **Secret** | Stores sensitive data (passwords, tokens) in base64-encoded form, injected into Pods. |
| **Volume** | A directory accessible to containers in a Pod. Can be ephemeral or persistent. |
| **PersistentVolume (PV)** | A piece of cluster-level storage provisioned by an admin or dynamically by a StorageClass. |
| **PersistentVolumeClaim (PVC)** | A user's request for storage. Binds to an available PV. |
| **StorageClass** | Defines a type of storage and how it's dynamically provisioned. |
| **ServiceAccount** | An identity assigned to Pods to interact with the Kubernetes API. |
| **RBAC** | Role-Based Access Control — controls who can do what within the cluster. |
| **Role / ClusterRole** | Defines a set of permissions. Role is namespace-scoped; ClusterRole is cluster-wide. |
| **RoleBinding / ClusterRoleBinding** | Grants a Role or ClusterRole to a user, group, or ServiceAccount. |
| **HPA** | HorizontalPodAutoscaler — automatically scales Pod replicas based on CPU/memory metrics. |
| **VPA** | VerticalPodAutoscaler — automatically adjusts CPU/memory requests/limits for containers. |
| **Affinity / Anti-Affinity** | Rules that attract or repel Pods from certain nodes or other Pods. |
| **Taint** | Marks a node so that Pods won't be scheduled on it unless they have a matching Toleration. |
| **Toleration** | Allows a Pod to be scheduled on a tainted node. |
| **Label** | Key-value pairs attached to objects for identification and selection. |
| **Annotation** | Key-value pairs for non-identifying metadata (e.g., build info, tool configs). |
| **Selector** | Used to filter/select objects by their labels. |
| **Probe** | Health check for a container: Liveness (is it alive?), Readiness (is it ready for traffic?), Startup. |
| **Resource Requests** | Minimum CPU/memory a container needs. Used for scheduling. |
| **Resource Limits** | Maximum CPU/memory a container can use. |
| **kube-proxy** | Runs on each node, maintains network rules to route traffic to the correct Pod. |
| **etcd** | Key-value store used by the Control Plane to persist all cluster state. |
| **API Server** | The front-end for the Kubernetes Control Plane. All commands go through it. |
| **Scheduler** | Assigns Pods to nodes based on resource requirements and constraints. |
| **Controller Manager** | Runs controllers (e.g., ReplicaSet, Deployment) that maintain desired state. |
| **kubelet** | Agent running on each node, ensures containers in Pods are running. |
| **Helm** | The package manager for Kubernetes. Installs pre-packaged apps called Charts. |
| **Chart** | A Helm package containing all K8s resource definitions for an application. |
| **Context** | A named combination of cluster, user, and namespace for kubectl. |
| **Manifest** | A YAML/JSON file describing a Kubernetes resource. |

---

## Architecture Overview

```
┌─────────────────────────────────────────────────┐
│                  CONTROL PLANE                  │
│  ┌──────────┐ ┌──────────┐ ┌─────────────────┐  │
│  │API Server│ │Scheduler │ │Controller Manager│  │
│  └──────────┘ └──────────┘ └─────────────────┘  │
│              ┌────────────┐                      │
│              │    etcd    │                      │
│              └────────────┘                      │
└─────────────────────────────────────────────────┘
           │               │
    ┌──────┴───┐     ┌──────┴───┐
    │  NODE 1  │     │  NODE 2  │
    │ ┌──────┐ │     │ ┌──────┐ │
    │ │kubelet│ │     │ │kubelet│ │
    │ ├──────┤ │     │ ├──────┤ │
    │ │ Pod  │ │     │ │ Pod  │ │
    │ │ Pod  │ │     │ │ Pod  │ │
    │ └──────┘ │     │ └──────┘ │
    │kube-proxy│     │kube-proxy│
    └──────────┘     └──────────┘
```

---

## kubectl — CLI Reference

> `kubectl` is the command-line tool to interact with your cluster.

### Cluster Info

```bash
kubectl version                          # Show client and server version
kubectl cluster-info                     # Display cluster endpoint info
kubectl get nodes                        # List all nodes
kubectl get nodes -o wide                # Nodes with extra details (IP, OS, etc.)
kubectl describe node <node-name>        # Detailed info about a node
kubectl top nodes                        # CPU/memory usage per node
```

### Namespaces

```bash
kubectl get namespaces                   # List all namespaces
kubectl create namespace <name>          # Create a namespace
kubectl delete namespace <name>          # Delete a namespace
kubectl config set-context --current --namespace=<name>   # Switch default namespace
```

### Pods

```bash
kubectl get pods                         # List pods in current namespace
kubectl get pods -A                      # List pods in all namespaces
kubectl get pods -o wide                 # Pods with node/IP info
kubectl get pod <pod-name> -o yaml       # Get full YAML of a pod
kubectl describe pod <pod-name>          # Detailed pod info + events
kubectl run <name> --image=<image>       # Run a pod imperatively
kubectl exec -it <pod-name> -- /bin/sh   # Shell into a running pod
kubectl exec -it <pod-name> -c <container> -- /bin/bash  # Shell into specific container
kubectl delete pod <pod-name>            # Delete a pod
kubectl delete pod <pod-name> --grace-period=0 --force   # Force delete
kubectl get pod <pod-name> -o jsonpath='{.status.podIP}' # Get Pod IP
```

### Deployments

```bash
kubectl get deployments                  # List deployments
kubectl create deployment <name> --image=<image>   # Create deployment
kubectl apply -f deployment.yaml         # Apply manifest
kubectl describe deployment <name>       # Describe deployment
kubectl delete deployment <name>         # Delete deployment
kubectl edit deployment <name>           # Edit deployment live
kubectl rollout status deployment/<name> # Check rollout status
kubectl rollout history deployment/<name>          # View rollout history
kubectl rollout undo deployment/<name>   # Rollback to previous version
kubectl rollout undo deployment/<name> --to-revision=2  # Rollback to specific revision
kubectl set image deployment/<name> <container>=<new-image>  # Update image
```

### Services

```bash
kubectl get services                     # List services
kubectl get svc                          # Short alias
kubectl describe service <name>          # Service details
kubectl expose deployment <name> --port=80 --type=ClusterIP    # Expose a deployment
kubectl expose deployment <name> --port=80 --type=NodePort
kubectl expose deployment <name> --port=80 --type=LoadBalancer
kubectl delete service <name>            # Delete a service
kubectl get endpoints <service-name>     # View endpoints behind a service
```

### ConfigMaps & Secrets

```bash
# ConfigMaps
kubectl get configmaps
kubectl create configmap <name> --from-literal=key=value
kubectl create configmap <name> --from-file=config.properties
kubectl describe configmap <name>
kubectl delete configmap <name>

# Secrets
kubectl get secrets
kubectl create secret generic <name> --from-literal=password=mysecret
kubectl create secret docker-registry <name> \
  --docker-server=<server> \
  --docker-username=<user> \
  --docker-password=<pass>
kubectl describe secret <name>
kubectl get secret <name> -o jsonpath='{.data.password}' | base64 --decode
```

### Persistent Volumes

```bash
kubectl get pv                           # List PersistentVolumes
kubectl get pvc                          # List PersistentVolumeClaims
kubectl describe pv <name>
kubectl describe pvc <name>
kubectl delete pvc <name>
```

### Ingress

```bash
kubectl get ingress                      # List ingress resources
kubectl describe ingress <name>
kubectl delete ingress <name>
```

### Logs & Debugging

```bash
kubectl logs <pod-name>                  # Logs of a pod
kubectl logs <pod-name> -c <container>   # Logs of specific container
kubectl logs <pod-name> --previous       # Logs of previously crashed container
kubectl logs <pod-name> -f               # Stream/follow logs
kubectl logs <pod-name> --tail=100       # Last 100 lines
kubectl logs -l app=myapp                # Logs from pods matching label

kubectl describe pod <pod-name>          # Events and status details
kubectl get events --sort-by=.metadata.creationTimestamp  # Recent events
kubectl get events -n <namespace>

kubectl port-forward pod/<pod-name> 8080:80    # Forward local port to pod
kubectl port-forward svc/<svc-name> 8080:80    # Forward local port to service

kubectl cp <pod-name>:/path/to/file ./local    # Copy file from pod
kubectl cp ./local <pod-name>:/path/to/file    # Copy file to pod

kubectl run debug --image=busybox -it --rm -- /bin/sh   # Ephemeral debug pod
kubectl debug -it <pod-name> --image=busybox --target=<container>  # Debug pod in-place
```

### Scaling & Rollouts

```bash
kubectl scale deployment <name> --replicas=5
kubectl autoscale deployment <name> --min=2 --max=10 --cpu-percent=80
kubectl get hpa                          # List HorizontalPodAutoscalers
kubectl rollout restart deployment/<name>          # Restart all pods in deployment
```

### Resource Management

```bash
kubectl apply -f <file.yaml>             # Create or update resources from file
kubectl apply -f ./directory/            # Apply all YAMLs in a directory
kubectl delete -f <file.yaml>            # Delete resources defined in file
kubectl diff -f <file.yaml>              # Preview changes before applying
kubectl replace -f <file.yaml>           # Replace a resource
kubectl patch deployment <name> -p '{"spec":{"replicas":3}}'  # Patch a resource

kubectl get all                          # Get all resource types in namespace
kubectl get all -A                       # Get all resources cluster-wide
kubectl api-resources                    # List all resource types
kubectl explain pod.spec                 # Documentation for a resource field
```

---

## YAML Manifests — Templates

### Pod Manifest

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
  namespace: default
  labels:
    app: my-app
spec:
  containers:
    - name: my-container
      image: nginx:latest
      ports:
        - containerPort: 80
      resources:
        requests:
          memory: "64Mi"
          cpu: "250m"
        limits:
          memory: "128Mi"
          cpu: "500m"
      env:
        - name: ENV_VAR
          value: "hello"
      livenessProbe:
        httpGet:
          path: /health
          port: 80
        initialDelaySeconds: 10
        periodSeconds: 5
      readinessProbe:
        httpGet:
          path: /ready
          port: 80
        initialDelaySeconds: 5
        periodSeconds: 3
```

### Deployment Manifest

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-deployment
  labels:
    app: my-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-app
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
        - name: my-container
          image: nginx:1.25
          ports:
            - containerPort: 80
          resources:
            requests:
              cpu: "100m"
              memory: "128Mi"
            limits:
              cpu: "500m"
              memory: "256Mi"
```

### Service Manifest

```yaml
# ClusterIP (internal only)
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  selector:
    app: my-app
  ports:
    - protocol: TCP
      port: 80
      targetPort: 8080
  type: ClusterIP   # ClusterIP | NodePort | LoadBalancer | ExternalName

---
# NodePort (external via node IP)
apiVersion: v1
kind: Service
metadata:
  name: my-nodeport-service
spec:
  selector:
    app: my-app
  ports:
    - port: 80
      targetPort: 8080
      nodePort: 30080     # Range: 30000–32767
  type: NodePort
```

### ConfigMap Manifest

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: my-config
data:
  APP_ENV: production
  LOG_LEVEL: info
  config.json: |
    {
      "key": "value"
    }
```

### Secret Manifest

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: my-secret
type: Opaque
data:
  username: dXNlcm5hbWU=    # base64 encoded
  password: cGFzc3dvcmQ=    # base64 encoded
```

> Encode: `echo -n 'mypassword' | base64`
> Decode: `echo 'cGFzc3dvcmQ=' | base64 --decode`

### Ingress Manifest

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
    - host: myapp.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: my-service
                port:
                  number: 80
  tls:
    - hosts:
        - myapp.example.com
      secretName: myapp-tls-secret
```

### PersistentVolumeClaim Manifest

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-pvc
spec:
  accessModes:
    - ReadWriteOnce       # ReadWriteOnce | ReadOnlyMany | ReadWriteMany
  resources:
    requests:
      storage: 5Gi
  storageClassName: standard
```

### HorizontalPodAutoscaler Manifest

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: my-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: my-deployment
  minReplicas: 2
  maxReplicas: 10
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
```

---

## Namespaces & Context Management

```bash
# Contexts
kubectl config get-contexts              # List all contexts
kubectl config current-context           # Show current context
kubectl config use-context <name>        # Switch context
kubectl config set-context <name> --cluster=<cluster> --user=<user> --namespace=<ns>

# Namespace shortcuts
kubectl get pods -n kube-system          # Query specific namespace
kubectl config set-context --current --namespace=dev   # Sticky namespace switch

# kubens and kubectx (popular tools)
# Install: https://github.com/ahmetb/kubectx
kubens <namespace>                       # Switch namespace
kubectx <context>                        # Switch context
```

---

## RBAC — Role-Based Access Control

```yaml
# Role (namespace-scoped)
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
  namespace: default
rules:
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["get", "list", "watch"]

---
# RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods-binding
  namespace: default
subjects:
  - kind: User
    name: jane
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

```bash
kubectl auth can-i get pods                       # Can current user get pods?
kubectl auth can-i get pods --as=jane             # Check as another user
kubectl auth can-i '*' '*'                        # Check if admin
kubectl get rolebindings -n <namespace>
kubectl get clusterrolebindings
```

---

## Networking Concepts

| Concept | Description |
|---------|-------------|
| **ClusterIP** | Default service type. Only accessible within the cluster. |
| **NodePort** | Exposes service on each node's IP at a static port (30000–32767). |
| **LoadBalancer** | Provisions an external load balancer (cloud providers). |
| **ExternalName** | Maps a service to an external DNS name. |
| **CNI Plugin** | Container Network Interface — handles Pod networking (e.g., Calico, Flannel, Cilium). |
| **DNS** | Every service gets a DNS name: `<service>.<namespace>.svc.cluster.local` |
| **NetworkPolicy** | Firewall rules for Pod-to-Pod and Pod-to-external communication. |

```yaml
# NetworkPolicy — deny all ingress, allow only from same namespace
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
spec:
  podSelector: {}
  policyTypes:
    - Ingress
```

---

## Storage Concepts

| Access Mode | Description |
|-------------|-------------|
| `ReadWriteOnce (RWO)` | Can be mounted read-write by a single node. |
| `ReadOnlyMany (ROX)` | Can be mounted read-only by many nodes. |
| `ReadWriteMany (RWX)` | Can be mounted read-write by many nodes. |

| Reclaim Policy | Description |
|----------------|-------------|
| `Retain` | PV is kept after PVC deletion. Must be manually reclaimed. |
| `Delete` | PV and underlying storage are deleted with the PVC. |
| `Recycle` | (Deprecated) Clears the volume for reuse. |

---

## Helm — Package Manager

```bash
# Installation
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

# Repo management
helm repo add stable https://charts.helm.sh/stable
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
helm repo list

# Searching
helm search repo nginx
helm search hub wordpress

# Installing charts
helm install <release-name> <chart>
helm install my-nginx bitnami/nginx
helm install my-app ./my-chart -f values.yaml
helm install my-app ./my-chart --set image.tag=v2.0

# Managing releases
helm list                               # List installed releases
helm list -A                            # All namespaces
helm status <release-name>
helm upgrade <release-name> <chart> -f values.yaml
helm upgrade --install <release> <chart>   # Install or upgrade
helm rollback <release-name> 1             # Rollback to revision 1
helm uninstall <release-name>

# Inspecting charts
helm show values bitnami/nginx          # Default values
helm template <release> <chart>         # Render templates locally
helm lint ./my-chart                    # Lint a chart

# Creating charts
helm create my-chart                    # Scaffold a new chart
```

---

## Common Patterns & Tips

**Force re-pull image in a Deployment:**
```bash
kubectl rollout restart deployment/<name>
```

**Run a one-off debug pod:**
```bash
kubectl run tmp --image=busybox --rm -it --restart=Never -- /bin/sh
```

**Watch pods in real time:**
```bash
kubectl get pods -w
```

**Get resource usage:**
```bash
kubectl top pods
kubectl top pods --sort-by=cpu
kubectl top pods --sort-by=memory
```

**Find what's crashing:**
```bash
kubectl get pods                        # Look for CrashLoopBackOff or Error
kubectl describe pod <pod>              # Check Events section
kubectl logs <pod> --previous           # Logs from last crashed instance
```

**Apply and dry-run:**
```bash
kubectl apply -f file.yaml --dry-run=client   # Validate locally
kubectl apply -f file.yaml --dry-run=server   # Validate on server
```

**Export a resource to YAML:**
```bash
kubectl get deployment <name> -o yaml > deployment.yaml
```

**Delete all pods matching a label:**
```bash
kubectl delete pods -l app=my-app
```

**Decode all secrets in a namespace:**
```bash
kubectl get secrets -o json | jq '.items[] | {name: .metadata.name, data: .data | map_values(@base64d)}'
```

---

## Glossary

| Term | Full Form / Definition |
|------|----------------------|
| **K8s** | Short for Kubernetes (8 letters between K and s) |
| **kubectl** | Kubernetes control — the CLI tool |
| **etcd** | Distributed key-value store for cluster state |
| **CNI** | Container Network Interface |
| **CRI** | Container Runtime Interface (e.g., containerd, CRI-O) |
| **CSI** | Container Storage Interface |
| **OCI** | Open Container Initiative — container standard |
| **HPA** | HorizontalPodAutoscaler |
| **VPA** | VerticalPodAutoscaler |
| **PV** | PersistentVolume |
| **PVC** | PersistentVolumeClaim |
| **RBAC** | Role-Based Access Control |
| **SA** | ServiceAccount |
| **CRD** | CustomResourceDefinition — extend K8s API with custom objects |
| **Operator** | A pattern using CRDs + controllers to manage complex apps |
| **Init Container** | Runs before app containers; used for setup tasks |
| **Sidecar** | An additional container in a Pod that augments the main container |
| **Affinity** | Rules for Pod scheduling preference on nodes or relative to other Pods |
| **Taint/Toleration** | Mechanism to repel/attract Pods to/from nodes |
| **QoS** | Quality of Service — Guaranteed, Burstable, or BestEffort based on resource settings |
| **kube-system** | Namespace for K8s internal components |
| **kubeconfig** | File (`~/.kube/config`) storing cluster access credentials and contexts |

---

## 🔗 Useful Resources

- [Official Kubernetes Docs](https://kubernetes.io/docs/)
- [kubectl Cheat Sheet (Official)](https://kubernetes.io/docs/reference/kubectl/cheatsheet/)
- [Helm Docs](https://helm.sh/docs/)
- [Kubernetes Playground (Killercoda)](https://killercoda.com/playgrounds/scenario/kubernetes)
- [K9s — Terminal UI for K8s](https://k9scli.io/)
- [kubectx + kubens](https://github.com/ahmetb/kubectx)

---

*Maintained with ❤️ — PRs and contributions welcome!*