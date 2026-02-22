# ☸️ Kubernetes — Command by Command
### Minimal · Precise · Notion Friendly

---

## 1. Create a Cluster

### With a config file

```bash
kind create cluster --config clusters.yml --name local
```

**`clusters.yml`** — the multi-node setup:

```yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
  - role: control-plane
  - role: worker
  - role: worker
```

| Part | What it does |
|---|---|
| `--config clusters.yml` | Tells kind to read node layout from this file |
| `--name local` | Names the cluster `local` (context becomes `kind-local`) |
| `control-plane` | One node that runs the API Server, Scheduler, etcd |
| `worker` (×2) | Two nodes that run your actual Pods |

> Each node = one Docker container on your machine.

---

### Without a yml file — inline flags

You don't need a yml file for simple setups. Just pass flags directly:

```bash
# Single node (control-plane only — simplest possible cluster)
kind create cluster --name local

# That's it. One command, zero files.
```

```bash
# With a specific Kubernetes version
kind create cluster --name local --image kindest/node:v1.33.1
```

> ⚠️ Single-node means the control-plane also acts as a worker. Fine for quick tests, not representative of a real cluster.

**No yml alternative for multi-node?**
There isn't a flag-only way to add worker nodes — `--config` is required for that. The workaround is to use the single-node default and accept the limitation, or always use a `clusters.yml`.

---

## 2. Check Your Nodes

```bash
kubectl get nodes
```

**What it does:** Asks the API Server for the list of all registered nodes and their status.

```
NAME                  STATUS   ROLES           AGE   VERSION
local-control-plane   Ready    control-plane   60s   v1.33.1
local-worker          Ready    <none>          45s   v1.33.1
local-worker2         Ready    <none>          45s   v1.33.1
```

| Column | Meaning |
|---|---|
| `STATUS: Ready` | Node is healthy, kubelet is running, can accept Pods |
| `STATUS: NotReady` | Node is down or kubelet isn't responding |
| `ROLES: control-plane` | Runs K8s internals, not your app Pods |
| `ROLES: <none>` | Worker node — runs your Pods |

```bash
# See also: which node each is on, the internal IP, OS, container runtime
kubectl get nodes -o wide
```

---

## 3. Delete the Cluster

```bash
kind delete cluster --name local
```

**What it does:** Stops and removes all Docker containers for this cluster. Also removes the cluster from your `~/.kube/config`.

> Nothing is saved. All pods, data, configs inside the cluster are gone.

---

## 4. Run nginx — Imperative (No YAML)

```bash
kubectl run nginx --image=nginx --port=80
```

**What it does:** Sends a request to the API Server to create a single bare Pod named `nginx` using the `nginx` Docker image.

| Part | What it does |
|---|---|
| `nginx` | The Pod name |
| `--image=nginx` | Pull the `nginx:latest` image from Docker Hub |
| `--port=80` | Documents that the container listens on port 80 (informational only — does NOT expose it externally) |

> This creates a **bare Pod** — not a Deployment. If it crashes, it stays dead. Good for quick tests only.

---

## 5. Check Pods

```bash
kubectl get pods
```

```
NAME    READY   STATUS    RESTARTS   AGE
nginx   1/1     Running   0          7s
```

| Column | Meaning |
|---|---|
| `READY 1/1` | 1 container running out of 1 total |
| `Running` | Container is alive and running |
| `RESTARTS: 0` | Never crashed |

**Status values you'll see:**

| Status | Meaning |
|---|---|
| `Pending` | Waiting to be scheduled onto a node |
| `ContainerCreating` | Image being pulled |
| `Running` | ✅ All good |
| `CrashLoopBackOff` | Container keeps crashing — check logs |
| `OOMKilled` | Ran out of memory |
| `ImagePullBackOff` | Can't find/pull the image |
| `Terminating` | Being deleted |

---

## 6. Read Pod Logs

```bash
kubectl logs nginx
```

**What it does:** Fetches all stdout/stderr output from the `nginx` container and prints it.

```
/docker-entrypoint.sh: Launching /docker-entrypoint.d/10-listen-on-ipv6-by-default.sh
/docker-entrypoint.sh: Configuration complete; ready for start up
```

> nginx prints access logs here every time someone hits it.

**The flag you actually want — stream live:**

```bash
kubectl logs -f nginx
```

`-f` = follow. Streams new log lines as they happen, like `tail -f`. Press `Ctrl+C` to stop.

```bash
# Last 50 lines only
kubectl logs --tail=50 nginx

# Logs from last 5 minutes
kubectl logs --since=5m nginx

# Combine: stream from last 5 minutes onward
kubectl logs --since=5m -f nginx
```

---

## 7. Delete a Pod

```bash
kubectl delete pod nginx
```

**What it does:** Sends a SIGTERM to the container, waits 30 seconds for graceful shutdown, then kills it. Removes the Pod object from K8s entirely.

```bash
kubectl get pods
# No resources found in default namespace.
```

> Since this was a bare Pod (no Deployment managing it), it's gone for good. Nothing recreates it.

---

## 8. Apply from a YAML File

```bash
kubectl apply -f manifest.yml
```

**What it does:** Reads the YAML file, sends its contents to the API Server, which creates or updates the resource(s) defined inside.

**Example `manifest.yml`:**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  labels:
    app: nginx
spec:
  containers:
    - name: nginx
      image: nginx:latest
      ports:
        - containerPort: 80
```

**`apply` vs `run` — key difference:**

| | `kubectl run` | `kubectl apply -f` |
|---|---|---|
| Style | Imperative — tell K8s what to do | Declarative — tell K8s what you want |
| Source of truth | Your terminal history | Your YAML file (version-controlled) |
| Re-run | Error if pod exists | Safe — updates if changed |
| Good for | Quick tests | Everything real |

```bash
# Delete using the same file
kubectl delete -f manifest.yml
```

> `delete -f` is cleaner than naming the pod — it deletes everything the file defines.

---

## 9. Watch Pods

```bash
kubectl get pods --watch
```

**What it does:** Keeps the connection open to the API Server and streams status changes as they happen. You see every state transition in real time.

```
NAME    READY   STATUS              RESTARTS   AGE
nginx   0/1     Pending             0          0s   ← being scheduled
nginx   0/1     ContainerCreating   0          1s   ← image pulling
nginx   1/1     Running             0          4s   ← ✅ up
```

If it crashes:
```
nginx   0/1     CrashLoopBackOff    1          10s  ← crashed, now check logs
```

> `--watch` watches **status changes**, not logs.
> To watch **logs**: `kubectl logs -f nginx`

```bash
# Shorthand
kubectl get pods -w

# Watch + see which node each pod is on
kubectl get pods -o wide --watch
```

---

## 10. Watch Individual Containers (from a multi-container YAML)

When your YAML defines multiple containers in one Pod, you need to be specific.

**Example YAML with 2 containers:**

```yaml
# manifest.yml
apiVersion: v1
kind: Pod
metadata:
  name: webapp
spec:
  containers:
    - name: nginx        # container 1
      image: nginx:latest
    - name: sidecar      # container 2
      image: busybox
      command: ["sh", "-c", "while true; do echo hello; sleep 5; done"]
```

```bash
kubectl apply -f manifest.yml
```

```bash
kubectl get pods
# NAME      READY   STATUS    RESTARTS   AGE
# webapp    2/2     Running   0          5s   ← 2/2 = both containers running
```

**Watch the overall pod status (both containers as one unit):**

```bash
kubectl get pods --watch
```

**Watch logs from a specific container:**

```bash
kubectl logs -f webapp -c nginx      # nginx container only
kubectl logs -f webapp -c sidecar    # sidecar container only
```

`-c` = container name (must match the `name:` field in your YAML).

**Check which containers are in a pod:**

```bash
kubectl describe pod webapp
# Lists every container, its state, restart count, and image
```

**See all container statuses at once:**

```bash
kubectl get pod webapp -o jsonpath='{range .status.containerStatuses[*]}{.name}{"\t"}{.state}{"\n"}{end}'
```

Or the readable way:

```bash
kubectl describe pod webapp | grep -A 5 "Containers:"
```

---

## Full Flow — Everything Together

```bash
# 1. Create cluster
kind create cluster --config clusters.yml --name local

# 2. Verify nodes are up
kubectl get nodes

# 3. Run nginx (quick imperative way)
kubectl run nginx --image=nginx --port=80

# 4. Check it started
kubectl get pods

# 5. Read its logs
kubectl logs nginx
kubectl logs -f nginx        # stream live

# 6. Delete it
kubectl delete pod nginx
kubectl get pods             # confirm gone

# 7. Apply from YAML (the real way)
kubectl apply -f manifest.yml

# 8. Watch it come up
kubectl get pods --watch

# 9. Watch logs of a specific container (multi-container pods)
kubectl logs -f <pod-name> -c <container-name>

# 10. Delete via file
kubectl delete pod nginx
# or cleaner:
kubectl delete -f manifest.yml

# 11. Tear down cluster
kind delete cluster --name local
```

---

## Quick Reference

```bash
# CLUSTER
kind create cluster --name local                          # single node, no file
kind create cluster --config clusters.yml --name local    # multi-node from file
kind delete cluster --name local                          # destroy everything

# NODES
kubectl get nodes                                         # list nodes + status
kubectl get nodes -o wide                                 # + IP, OS, runtime

# PODS — create
kubectl run nginx --image=nginx --port=80                 # imperative (no yaml)
kubectl apply -f manifest.yml                             # declarative (yaml)

# PODS — view
kubectl get pods                                          # snapshot
kubectl get pods -w                                       # live status stream
kubectl get pods -o wide                                  # + node + IP

# PODS — logs
kubectl logs nginx                                        # all logs so far
kubectl logs -f nginx                                     # stream live ← use this
kubectl logs --tail=50 nginx                              # last 50 lines
kubectl logs -f nginx -c <container>                      # specific container

# PODS — delete
kubectl delete pod nginx                                  # by name
kubectl delete -f manifest.yml                            # by file
```