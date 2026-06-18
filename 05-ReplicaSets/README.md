# ReplicaSets

> A ReplicaSet keeps a specified number of identical pods running.
> If a pod dies, it creates a replacement automatically (self-healing).
> Rarely used directly — Deployments manage ReplicaSets for you.

---

## Goal
Understand how ReplicaSets keep pods alive, self-heal, and scale.

---

## How It Works
[ReplicaSet] → "always keep 3 pods"

├── [Pod]

├── [Pod]

└── [Pod]   → if one dies, a new one is created instantly

You declare the DESIRED state (replicas: 3), and the ReplicaSet
constantly works to match reality to that number.

---

## Files
- `replicaset.yaml` — ReplicaSet keeping 3 vproapp pods

---

## Steps

### 1. Write the definition
```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: vproapp-rs
  labels:
    app: vproapp
spec:
  replicas: 3
  selector:
    matchLabels:
      app: vproapp
  template:
    metadata:
      labels:
        app: vproapp
    spec:
      containers:
        - name: appcontainer
          image: nginx:1.25
          ports:
            - containerPort: 80
```

### 2. Apply it
```bash
kubectl apply -f replicaset.yaml
```

### 3. See the pods created
```bash
kubectl get pods
# 3 pods named vproapp-rs-xxxxx

kubectl get rs
# NAME         DESIRED   CURRENT   READY   AGE
# vproapp-rs   3         3         3       10s
```

### 4. Test self-healing — delete a pod
```bash
kubectl delete pod vproapp-rs-xxxxx
kubectl get pods
# still 3 — ReplicaSet created a replacement automatically
```

### 5. Scale it
```bash
kubectl scale replicaset vproapp-rs --replicas=5
kubectl get pods   # now 5
```

### 6. Clean up
```bash
kubectl delete -f replicaset.yaml
```

---

## What I Learned
- ReplicaSet's only job is keeping N pods running (self-healing)
- selector.matchLabels must match template.metadata.labels, or it errors
- template is just a Pod definition embedded inside the ReplicaSet
- deleting a pod under a ReplicaSet → it's instantly recreated
- this demonstrates Kubernetes' declarative model: you declare desired
  state, k8s continuously reconciles reality to match it
- ReplicaSet has NO rolling update or rollback — that's why Deployments exist
- in real use you create Deployments, not ReplicaSets directly

---

## Gotchas & Mistakes
- (add issues as you encounter them)
- common one: selector labels not matching template labels → ReplicaSet rejected

---

## Cheatsheet

| Command | What it does |
|---|---|
| `kubectl get rs` | list replicasets and their desired/current/ready counts |
| `kubectl describe rs <name>` | detailed info and events |
| `kubectl scale rs <name> --replicas=N` | change number of pods |
| `kubectl delete pod <name>` | delete a pod (watch it get recreated) |
| `kubectl delete -f replicaset.yaml` | delete the replicaset and its pods |

---

## Next Section
→ [Deployments](../06-deployments/README.md)
