# 🌐 Ingress & Ingress Controller — Complete Guide

> **Official Docs:** https://kubernetes.io/docs/concepts/services-networking/ingress/

---

## ⚠️ Downsides of LoadBalancer Services

Before understanding Ingress, let's understand **why LoadBalancer services alone are not enough** for production systems.

### Problem 1: One Load Balancer Per Service

If you have three apps — frontend, backend, and a WebSocket server — you must create **3 separate LoadBalancer services**.

```
Internet
   │
   ├──► LoadBalancer 1 (IP: 34.10.1.1)  ──► frontend Pods
   ├──► LoadBalancer 2 (IP: 34.10.1.2)  ──► backend Pods
   └──► LoadBalancer 3 (IP: 34.10.1.3)  ──► websocket Pods
```

**Problems:**
- **No centralized traffic management** — you can't route `myapp.com/api` → backend and `myapp.com/` → frontend with a single entry point
- **No path-based routing** — every service gets its own external IP
- **Cloud provider limits** — AWS, GCP, Azure all have **quotas** on how many load balancers your account can create
- **Cost** — Each cloud load balancer costs money. 3 apps = 3x the cost

```bash
# What this looks like — 3 services, 3 external IPs, 3 bills
kubectl get services

# NAME               TYPE           CLUSTER-IP     EXTERNAL-IP    PORT(S)
# frontend-svc       LoadBalancer   10.0.0.1       34.10.1.1      80:30001/TCP
# backend-svc        LoadBalancer   10.0.0.2       34.10.1.2      80:30002/TCP
# websocket-svc      LoadBalancer   10.0.0.3       34.10.1.3      80:30003/TCP
```

---

### Problem 2: No Centralized TLS/SSL Certificate Management

```
┌──────────────────────────────────────────────────────┐
│  Without Ingress — SSL Hell                          │
│                                                      │
│  LB 1: cert for frontend.myapp.com  ← manual setup  │
│  LB 2: cert for api.myapp.com       ← manual setup  │
│  LB 3: cert for ws.myapp.com        ← manual setup  │
│                                                      │
│  3 certs × N services = N manual renewals per year  │
└──────────────────────────────────────────────────────┘
```

**Problems:**
- Certificates must be **managed outside the cluster** and attached manually to each load balancer
- When a cert expires, you must **manually rotate** each one
- No single place to manage TLS termination — it's scattered across every service
- No support for **Let's Encrypt auto-renewal** built into the load balancer

---

### Problem 3: No Centralized Rate Limiting

```
┌──────────────────────────────────────────────────┐
│  Without Ingress — Scattered Rate Limits         │
│                                                  │
│  LB 1 → frontend: rate limit = 1000 req/min     │
│  LB 2 → backend:  rate limit = 500 req/min      │
│  LB 3 → ws:       rate limit = ???               │
│                                                  │
│  No way to apply ONE global rate limit policy    │
└──────────────────────────────────────────────────┘
```

**Problems:**
- Each load balancer has its own configuration — **no unified policy**
- You can't apply a **single rate limit** across all services
- No global DDoS protection layer
- Configuration drift — each LB configured differently by different people

---

### Summary: Why LoadBalancer Alone Falls Short

| Feature | LoadBalancer Service | Ingress |
|---|---|---|
| Path-based routing (`/api`, `/`) | ❌ No | ✅ Yes |
| Host-based routing (subdomains) | ❌ No | ✅ Yes |
| Single external IP for all services | ❌ One IP per service | ✅ One IP for all |
| Centralized TLS/SSL | ❌ Per service | ✅ One place |
| Rate limiting across services | ❌ Per service | ✅ Global |
| Cost efficiency | ❌ $$ per LB | ✅ $ one LB |
| Auth/Auth middleware | ❌ No | ✅ Annotations |

---

## 🚪 What is Ingress?

**Ingress** is a Kubernetes API object that manages **external HTTP/HTTPS access** to services inside the cluster.

> Think of Ingress as a **smart router/reverse proxy** that sits in front of all your services and decides where each request should go.

```
                                   Kubernetes Cluster
Internet                          ┌──────────────────────────────────────┐
                                  │                                      │
  Users ──► Single LB ──► Ingress │  /        ──► frontend-service       │
            (1 IP)       Controller│  /api     ──► backend-service        │
                                  │  /ws      ──► websocket-service      │
                                  │  myapp.com ─► frontend-service       │
                                  │  api.myapp.com ─► backend-service    │
                                  └──────────────────────────────────────┘
```

### What Ingress Can Do
- ✅ **Load balancing** — distribute traffic across pod replicas
- ✅ **SSL/TLS termination** — handle HTTPS, pass HTTP to pods internally
- ✅ **Name-based virtual hosting** — different domains → different services
- ✅ **Path-based routing** — `/api` → backend, `/` → frontend
- ✅ **Middleware** — auth, rate limiting, CORS (via annotations)

> 💡 **Important:** Ingress only works for **HTTP and HTTPS**. For TCP/UDP protocols (e.g., databases, custom protocols), you still need `NodePort` or `LoadBalancer`.

---

## 🧩 Ingress vs Ingress Controller — What's the Difference?

| | Ingress | Ingress Controller |
|---|---|---|
| **What** | A Kubernetes resource (YAML config) | A running software (Pod/Deployment) |
| **Role** | Defines routing rules | Actually implements those rules |
| **Ships with K8s?** | ✅ Yes (as an API type) | ❌ No — must install manually |
| **Analogy** | Traffic routing rulebook | The actual traffic cop enforcing rules |

**Without an Ingress Controller, Ingress rules do NOTHING.**

---

## ⚙️ The Ingress Controller

### What is it?

The **Ingress Controller** is a Pod running inside your cluster that:
1. Watches for `Ingress` resource changes via the Kubernetes API
2. Reads the routing rules you define
3. Configures its underlying proxy (NGINX, HAProxy, Traefik, etc.) accordingly
4. Routes incoming traffic to the correct service

```
┌───────────────────────────────────────────────────────────────┐
│                     Kubernetes Cluster                        │
│                                                               │
│  ┌──────────────────────────────────────┐                     │
│  │         Control Plane                │                     │
│  │  API Server ← Ingress Controller     │                     │
│  │              (watches Ingress CRs)   │                     │
│  └──────────────────────────────────────┘                     │
│                        │                                      │
│                        ▼ reads rules, configures proxy        │
│  ┌──────────────────────────────────────┐                     │
│  │     Ingress Controller Pod           │                     │
│  │     (e.g. NGINX running inside)      │                     │
│  │                                      │                     │
│  │  /         → frontend-svc:80         │                     │
│  │  /api      → backend-svc:3000        │                     │
│  │  /ws       → ws-svc:8080             │                     │
│  └──────────────────────────────────────┘                     │
└───────────────────────────────────────────────────────────────┘
```

### Popular Ingress Controllers

| Controller | Underlying Proxy | Best For |
|---|---|---|
| **NGINX Ingress Controller** | NGINX | Most popular, general purpose |
| **Traefik** | Traefik | Dynamic config, Let's Encrypt auto TLS |
| **HAProxy Ingress** | HAProxy | High performance |
| **AWS ALB Ingress** | AWS ALB | AWS-native, integrates with IAM |
| **GCE Ingress** | Google Cloud LB | GKE-native |
| **Kong Ingress** | Kong | API Gateway features, plugins |
| **Istio Gateway** | Envoy | Service mesh, advanced traffic management |

Full list: https://kubernetes.io/docs/concepts/services-networking/ingress-controllers/

---

### Installing NGINX Ingress Controller

```bash
# Option 1: Using kubectl (bare metal / kind / minikube)
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.9.4/deploy/static/provider/cloud/deploy.yaml

# Option 2: Using Helm (recommended for production)
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update

helm install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx \
  --create-namespace

# For Minikube specifically:
minikube addons enable ingress

# For kind (Kubernetes IN Docker):
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.9.4/deploy/static/provider/kind/deploy.yaml

# Verify it's running
kubectl get pods -n ingress-nginx
kubectl get svc -n ingress-nginx

# You should see:
# NAME                                 TYPE           CLUSTER-IP    EXTERNAL-IP
# ingress-nginx-controller             LoadBalancer   10.96.0.1     <EXTERNAL-IP>
```

---

## 📐 Ingress Types

### Type 1: Simple Default Backend

Routes **all traffic** to a single service — a catch-all fallback.

```yaml
# simple-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: simple-ingress
  namespace: default
spec:
  ingressClassName: nginx
  defaultBackend:
    service:
      name: frontend-service
      port:
        number: 80
```

```bash
kubectl apply -f simple-ingress.yaml
kubectl get ingress
kubectl describe ingress simple-ingress
```

---

### Type 2: Path-Based Routing (Fan Out)

Route traffic from a **single host** to multiple services based on the URL path.

```
myapp.com/         ──► frontend-service:80
myapp.com/api      ──► backend-service:3000
myapp.com/static   ──► static-service:80
```

```yaml
# path-based-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: path-based-ingress
  namespace: default
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /   # Strip path prefix before forwarding
spec:
  ingressClassName: nginx
  rules:
    - host: myapp.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: frontend-service
                port:
                  number: 80
          - path: /api
            pathType: Prefix
            backend:
              service:
                name: backend-service
                port:
                  number: 3000
          - path: /static
            pathType: Prefix
            backend:
              service:
                name: static-service
                port:
                  number: 80
```

**`pathType` Options:**
- `Prefix` — matches the path prefix (e.g., `/api` matches `/api`, `/api/users`, `/api/v2/data`)
- `Exact` — matches only the exact path (e.g., `/api` matches only `/api`, NOT `/api/users`)
- `ImplementationSpecific` — behavior depends on the Ingress controller

```bash
kubectl apply -f path-based-ingress.yaml

# Test routing
curl http://myapp.com/          # → frontend
curl http://myapp.com/api       # → backend
curl http://myapp.com/static    # → static server
```

---

### Type 3: Host-Based Routing (Name-Based Virtual Hosting)

Route traffic based on the **hostname/subdomain** — one IP, multiple domains.

```
myapp.com          ──► frontend-service
api.myapp.com      ──► backend-service
ws.myapp.com       ──► websocket-service
```

```yaml
# host-based-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: host-based-ingress
  namespace: default
spec:
  ingressClassName: nginx
  rules:
    - host: myapp.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: frontend-service
                port:
                  number: 80

    - host: api.myapp.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: backend-service
                port:
                  number: 3000

    - host: ws.myapp.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: websocket-service
                port:
                  number: 8080
```

```bash
kubectl apply -f host-based-ingress.yaml

# Test (ensure DNS or /etc/hosts points these to Ingress IP)
curl http://myapp.com
curl http://api.myapp.com
curl http://ws.myapp.com
```

---

### Type 4: TLS / HTTPS Ingress

Terminate SSL at the Ingress and forward plain HTTP internally to services.

```
User ──(HTTPS)──► Ingress (TLS termination) ──(HTTP)──► backend Pod
                   ↑
              cert stored as K8s Secret
```

**Step 1: Create a TLS Secret**

```bash
# Option A: Self-signed cert (for testing)
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout tls.key \
  -out tls.crt \
  -subj "/CN=myapp.com/O=myapp"

# Create the Kubernetes secret
kubectl create secret tls myapp-tls-secret \
  --key tls.key \
  --cert tls.crt

# Option B: Let's Encrypt (production) — requires cert-manager
# Install cert-manager first:
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.13.0/cert-manager.yaml
```

**Step 2: Create TLS Ingress**

```yaml
# tls-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: tls-ingress
  namespace: default
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"     # Force HTTP → HTTPS redirect
    cert-manager.io/cluster-issuer: "letsencrypt-prod"   # Auto cert (cert-manager)
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - myapp.com
        - api.myapp.com
      secretName: myapp-tls-secret          # The secret holding the cert
  rules:
    - host: myapp.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: frontend-service
                port:
                  number: 80
    - host: api.myapp.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: backend-service
                port:
                  number: 3000
```

```bash
kubectl apply -f tls-ingress.yaml

# Verify TLS
kubectl describe ingress tls-ingress
kubectl get secret myapp-tls-secret
```

---

## 🏗️ Full Real-World Example

### Scenario: E-commerce App with 3 Services

```
ecommerce.com/           ──► store-frontend  (port 80)
ecommerce.com/api        ──► store-backend   (port 3000)
ecommerce.com/admin      ──► admin-panel     (port 8080)
```

**Step 1: Create Deployments**

```bash
# Frontend
kubectl create deployment store-frontend --image=nginx:latest
kubectl expose deployment store-frontend --port=80

# Backend API
kubectl create deployment store-backend --image=node:18
kubectl expose deployment store-backend --port=3000

# Admin panel
kubectl create deployment admin-panel --image=nginx:latest
kubectl expose deployment admin-panel --port=8080
```

**Step 2: Create the Ingress**

```yaml
# ecommerce-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ecommerce-ingress
  namespace: default
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /$2
    nginx.ingress.kubernetes.io/ssl-redirect: "false"
spec:
  ingressClassName: nginx
  rules:
    - host: ecommerce.com
      http:
        paths:
          - path: /api(/|$)(.*)
            pathType: ImplementationSpecific
            backend:
              service:
                name: store-backend
                port:
                  number: 3000
          - path: /admin(/|$)(.*)
            pathType: ImplementationSpecific
            backend:
              service:
                name: admin-panel
                port:
                  number: 8080
          - path: /()(.*)
            pathType: ImplementationSpecific
            backend:
              service:
                name: store-frontend
                port:
                  number: 80
```

**Step 3: Apply and Verify**

```bash
kubectl apply -f ecommerce-ingress.yaml

# Get Ingress status and external IP
kubectl get ingress
kubectl describe ingress ecommerce-ingress

# For local testing — add to /etc/hosts:
# <INGRESS-EXTERNAL-IP>   ecommerce.com
echo "127.0.0.1 ecommerce.com" | sudo tee -a /etc/hosts

# Test
curl http://ecommerce.com/          # → frontend
curl http://ecommerce.com/api       # → backend
curl http://ecommerce.com/admin     # → admin panel
```

---

## 🛡️ Useful Ingress Annotations (NGINX)

Annotations let you configure advanced features on your Ingress:

```yaml
metadata:
  annotations:
    # Rate limiting — max 100 requests per minute per IP
    nginx.ingress.kubernetes.io/limit-rps: "100"
    nginx.ingress.kubernetes.io/limit-connections: "10"

    # Force HTTPS redirect
    nginx.ingress.kubernetes.io/ssl-redirect: "true"

    # CORS headers
    nginx.ingress.kubernetes.io/enable-cors: "true"
    nginx.ingress.kubernetes.io/cors-allow-origin: "https://myapp.com"

    # Timeout settings
    nginx.ingress.kubernetes.io/proxy-connect-timeout: "60"
    nginx.ingress.kubernetes.io/proxy-send-timeout: "60"
    nginx.ingress.kubernetes.io/proxy-read-timeout: "60"

    # Max upload size
    nginx.ingress.kubernetes.io/proxy-body-size: "50m"

    # Basic auth
    nginx.ingress.kubernetes.io/auth-type: basic
    nginx.ingress.kubernetes.io/auth-secret: basic-auth-secret
    nginx.ingress.kubernetes.io/auth-realm: "Authentication Required"

    # Rewrite target path
    nginx.ingress.kubernetes.io/rewrite-target: /
```

---

## 🔍 Debugging Ingress

```bash
# Check if Ingress was created and has an IP
kubectl get ingress
kubectl get ingress -A    # All namespaces

# Detailed info — rules, backends, TLS
kubectl describe ingress <ingress-name>

# Check Ingress Controller logs (where routing actually happens)
kubectl logs -n ingress-nginx -l app.kubernetes.io/name=ingress-nginx --tail=100

# Follow logs in real time
kubectl logs -n ingress-nginx -l app.kubernetes.io/name=ingress-nginx -f

# Check Ingress Controller Pod is running
kubectl get pods -n ingress-nginx

# Check Ingress Controller Service has external IP
kubectl get svc -n ingress-nginx

# Common issues:
# 1. No EXTERNAL-IP → Ingress controller not installed / cloud LB pending
# 2. 404 on all routes → Check path rules and service names
# 3. 503 → Service exists but no healthy pods backing it
# 4. TLS errors → Secret name mismatch or cert expired
```

---

## 🗺️ Architecture Recap

```
                            ┌─────────────────────────────────────────────┐
                            │              Kubernetes Cluster              │
                            │                                             │
Internet                    │  ┌──────────────────────────────────────┐   │
   │                        │  │         NGINX Ingress Controller     │   │
Users ──► DNS ──► (1 LB IP) ──►│  Reads Ingress rules from API Server │   │
                            │  │                                      │   │
                            │  │  Rule: /       → frontend-svc:80    │   │
                            │  │  Rule: /api    → backend-svc:3000   │   │
                            │  │  Rule: /admin  → admin-svc:8080     │   │
                            │  └──────────────────────────────────────┘   │
                            │          │            │           │          │
                            │          ▼            ▼           ▼          │
                            │   ┌─────────┐  ┌─────────┐ ┌─────────┐    │
                            │   │frontend │  │ backend │ │  admin  │    │
                            │   │  Pods   │  │  Pods   │ │  Pods   │    │
                            │   └─────────┘  └─────────┘ └─────────┘    │
                            └─────────────────────────────────────────────┘
```

---

## 📋 Quick Command Reference

```bash
# Install NGINX Ingress Controller
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.9.4/deploy/static/provider/cloud/deploy.yaml

# Minikube shortcut
minikube addons enable ingress

# List all ingress resources
kubectl get ingress
kubectl get ingress -A

# Describe an ingress
kubectl describe ingress <name>

# Delete ingress
kubectl delete ingress <name>
kubectl delete -f ingress.yaml

# Check controller logs
kubectl logs -n ingress-nginx deploy/ingress-nginx-controller

# Get Ingress controller external IP
kubectl get svc ingress-nginx-controller -n ingress-nginx

# Apply ingress manifest
kubectl apply -f ingress.yaml

# Test locally with /etc/hosts
sudo bash -c 'echo "$(kubectl get svc -n ingress-nginx ingress-nginx-controller -o jsonpath="{.status.loadBalancer.ingress[0].ip}") myapp.com" >> /etc/hosts'
```

---

## 🔗 Useful Resources

- [Official Ingress Docs](https://kubernetes.io/docs/concepts/services-networking/ingress/)
- [Ingress Controllers List](https://kubernetes.io/docs/concepts/services-networking/ingress-controllers/)
- [NGINX Ingress Controller Docs](https://kubernetes.github.io/ingress-nginx/)
- [Traefik Kubernetes Docs](https://doc.traefik.io/traefik/providers/kubernetes-ingress/)
- [cert-manager (auto TLS)](https://cert-manager.io/docs/)

---

*Part of the Kubernetes Learning Series ☸️*
