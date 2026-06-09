# Kubernetes Learning Roadmap

## Progress
- [x] 01 - Cluster Setup
- [ ] 02 - Deployments
- [ ] 03 - Networking
- [ ] 04 - Storage

---

## 01 — Cluster Setup
**Goal:** Get a local cluster running with kubeadm or minikube

### Steps
1. Install kubectl
```bash
   curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
   chmod +x kubectl && mv kubectl /usr/local/bin/
```

2. Start minikube
```bash
   minikube start --driver=docker
```

3. Verify cluster is running
```bash
   kubectl get nodes
   # Expected output: node with STATUS = Ready
```

### What I Learned
- kubectl is the CLI that talks to the cluster API
- minikube runs a single-node cluster locally inside Docker
- kubeconfig at ~/.kube/config tells kubectl which cluster to talk to

### Gotchas / Mistakes I Made
- Forgot to start Docker before running minikube → got a driver error
- Used wrong context, commands were hitting old cluster → fix: `kubectl config use-context minikube`

---

## 02 — Deployments
**Goal:** Deploy an app and understand how Pods, ReplicaSets and Deployments relate

### Steps
1. Write a deployment manifest
2. Apply it → `kubectl apply -f deployment.yaml`
3. Check rollout → `kubectl rollout status deployment/myapp`
4. Scale it → `kubectl scale deployment myapp --replicas=3`

### What I Learned
- A Deployment manages ReplicaSets which manage Pods
- If you delete a Pod manually, the ReplicaSet recreates it automatically

### Gotchas / Mistakes I Made
- Set wrong container port in the manifest, app wasn't reachable
- Used `kubectl create` instead of `kubectl apply` → can't re-run idempotently
