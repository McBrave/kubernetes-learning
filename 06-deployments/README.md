# Deployments

> A Deployment manages ReplicaSets (which manage Pods) and adds the two
> things that matter most in real operations: rolling updates (change
> versions with zero downtime) and rollbacks (instantly undo a bad release).
> This is the object you actually use for almost every app in real life.

---

## Goal
Understand how Deployments manage pods through ReplicaSets,
and how rolling updates and rollbacks work.

---

## How It Works
[Deployment]           ← you create this

│ manages

[ReplicaSet]           ← created automatically

│ manages

[Pod] [Pod] [Pod]      ← created automatically

You only manage the Deployment — it handles the ReplicaSet and Pods underneath.

---

## Files
- `deployment.yaml` — Deployment running 3 vproapp pods

---

## Steps

### 1. Write the definition
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: vproapp-deploy
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

> almost identical to a ReplicaSet — only difference is kind: Deployment
> that one change unlocks rolling updates + rollbacks

### 2. Apply it
```bash
kubectl apply -f deployment.yaml
```

### 3. See what it created
```bash
kubectl get deployments
kubectl get rs      # the ReplicaSet it made automatically
kubectl get pods    # the 3 pods
```

### 4. Scale it
```bash
kubectl scale deployment vproapp-deploy --replicas=5
kubectl get pods    # now 5
```

### 5. Rolling update — change the image
```bash
# method 1: edit image in yaml to nginx:1.26, then
kubectl apply -f deployment.yaml

# method 2: direct command
kubectl set image deployment/vproapp-deploy appcontainer=nginx:1.26

# watch the rolling update happen live
kubectl rollout status deployment/vproapp-deploy
```

> pods are replaced gradually, not all at once → zero downtime

### 6. View revision history
```bash
kubectl rollout history deployment/vproapp-deploy
```

### 7. Rollback if a release breaks
```bash
# undo to previous version
kubectl rollout undo deployment/vproapp-deploy

# rollback to a specific revision
kubectl rollout undo deployment/vproapp-deploy --to-revision=2
```

### 8. Clean up
```bash
kubectl delete -f deployment.yaml
```

---

## How Deployment + Service Work Together
Deployment → creates pods stamped with label app: vproapp

Service    → finds pods with label app: vproapp, routes traffic to them
They never reference each other directly — the shared LABEL is the only link.
This is why rolling updates work seamlessly: the Service keeps routing to
whatever currently has the matching label, even as pods get replaced.

---

## What I Learned
- Deployment yaml is nearly identical to ReplicaSet — just kind: Deployment
- Deployment automatically creates and manages a ReplicaSet for you
- rolling update replaces pods gradually → zero downtime, unlike the
  manual delete/recreate needed for raw Pods
- Deployment keeps old ReplicaSets around → enables instant rollback
- rollback works by scaling an old ReplicaSet back up
- Service connects to pods via labels, NOT to the Deployment directly
- in real life you create Deployments, never bare Pods or ReplicaSets
- if no namespace is set, Deployment + its pods go into "default" namespace
- the ReplicaSet and Pods inherit the Deployment's namespace automatically

---

## Gotchas & Mistakes
- selector.matchLabels must match template.metadata.labels or it errors
- Service selector must match the pod labels, or it finds zero pods
  → check with kubectl get endpoints <service-name>
- Service and Deployment must be in the SAME namespace to connect
- (add more as you encounter them)

---

## Cheatsheet

| Command | What it does |
|---|---|
| `kubectl get deployments` | list deployments |
| `kubectl get rs` | see the ReplicaSets deployments created |
| `kubectl describe deployment <name>` | detailed info and events |
| `kubectl scale deployment <name> --replicas=N` | change pod count |
| `kubectl set image deployment/<name> <container>=<image>` | update image |
| `kubectl rollout status deployment/<name>` | watch a rollout progress |
| `kubectl rollout history deployment/<name>` | see revision history |
| `kubectl rollout undo deployment/<name>` | rollback to previous version |
| `kubectl rollout undo deployment/<name> --to-revision=N` | rollback to specific revision |
| `kubectl get endpoints <service-name>` | see which pods a service routes to |

---

## Next Section
→ [ConfigMaps & Secrets](../07-configmaps-secrets/README.md)
