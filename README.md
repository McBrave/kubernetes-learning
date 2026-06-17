# Kubernetes Learning Roadmap

## Progress
- [x] 01 - Cluster Setup
  - [x] minikube — local cluster
  - [x] kops — AWS cluster
- [x] 02 - Pod Definitions
- [ ] 03 - Deployments
- [ ] 04 - Networking
- [ ] 05 - Storage

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
