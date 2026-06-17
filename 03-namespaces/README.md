# Namespaces

> A namespace is a logical grouping inside one cluster —
> not a separate cluster, not separate hardware.
> Used to isolate and organize resources (pods, services, deployments).

---

## Goal
Understand what namespaces are, why they exist,
and how to use them with kubectl.

---

## How It Works
[One Cluster]

│

├── [Namespace: dev]

├── [Namespace: staging]

├── [Namespace: production]

└── [Namespace: default]      ← used if none specified

---

## Default Namespaces Already in Any Cluster
```bash
kubectl get namespaces

# NAME              STATUS   AGE
# default           Active   10d
# kube-system        Active   10d   ← k8s internal components
# kube-public        Active   10d   ← publicly readable info
# kube-node-lease    Active   10d   ← node heartbeat tracking
```

---

## Steps

### 1. Create a namespace
```bash
kubectl create namespace dev
```

### 2. Create a resource inside it — two ways

via flag:
```bash
kubectl apply -f vproapppod.yaml -n dev
```

via yaml:
```yaml
metadata:
  name: vproapp
  namespace: dev
```

### 3. View resources per namespace
```bash
kubectl get pods                  # default namespace only
kubectl get pods -n dev           # specific namespace
kubectl get pods -A               # all namespaces
```

### 4. Set a default namespace (avoid typing -n every time)
```bash
kubectl config set-context --current --namespace=dev

# verify
kubectl config view --minify | grep namespace
```

### 5. Delete a namespace
```bash
kubectl delete namespace dev
```
> ⚠️ deletes everything inside it — all pods, services, deployments

---

## What I Learned
- namespace = logical separation inside one cluster, not physical isolation
- same resource name can exist in multiple namespaces without conflict
- if a pod is "missing" it's often just in a different namespace —
  check with kubectl get pods -A first
- you've been using the default namespace this whole time without realizing it
- setting a default namespace via set-context saves typing -n constantly

---

## Gotchas & Mistakes
- created a pod, ran kubectl get pods, saw nothing → pod was in
  a different namespace, fixed by checking kubectl get pods -A
- (add more as you encounter them)

---

## Cheatsheet

| Command | What it does |
|---|---|
| `kubectl get namespaces` | list all namespaces |
| `kubectl create namespace <name>` | create new namespace |
| `kubectl delete namespace <name>` | delete namespace + everything inside |
| `kubectl get pods -n <name>` | pods in specific namespace |
| `kubectl get pods -A` | pods across all namespaces |
| `kubectl config set-context --current --namespace=<name>` | set default namespace |
| `kubectl config view --minify \| grep namespace` | check current default namespace |

---

## Next Section
→ [Deployments](../04-deployments/README.md)
