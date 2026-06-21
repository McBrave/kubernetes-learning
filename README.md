# Kubernetes Learning Roadmap

> My hands-on journey learning Kubernetes — structured notes, working yaml files,
> and real mistakes documented along the way.
> Following a course using kops on EC2, but running most things locally
> on Docker Desktop's built-in Kubernetes (kind).

---

## Progress
- [x] 01 - Cluster Setup
  - [x] minikube — local cluster
  - [x] kops — AWS cluster
- [x] 02 - Pod Definitions
- [x] 03 - Namespaces
- [x] 04 - Services
- [x] 05 - ReplicaSets
- [x] 06 - Deployments
- [ ] 07 - ConfigMaps & Secrets

---

## Sections

### 01 — Cluster Setup
Setting up a Kubernetes cluster, both locally and on AWS.
- [minikube](01-cluster-setup/minikube/README.md) — local single-node cluster
- [kops](01-cluster-setup/kops/README.md) — production-like cluster on AWS

### 02 — Pod Definitions
[→ details](02-pod-definitions/README.md)
Raw Pod creation with yaml, kubectl basics, and why bare pods are limited.

### 03 — Namespaces
[→ details](03-namespaces/README.md)
Logical isolation inside one cluster — grouping and separating resources.

### 04 — Services
[→ details](04-services/README.md)
Stable network access for pods. ClusterIP, NodePort, and LoadBalancer types.

### 05 — ReplicaSets
[→ details](05-replicasets/README.md)
Keeping N identical pods alive with self-healing and scaling.

### 06 — Deployments
[→ details](06-deployments/README.md)
Managing ReplicaSets with rolling updates and rollbacks. The object used in real life.

### 07 — ConfigMaps & Secrets
*(in progress)*

---

## Key Concepts So Far
Pod          → single instance of the app

ReplicaSet   → keeps N identical pods alive (self-healing)

Deployment   → manages ReplicaSets + rolling updates + rollback

Service      → stable network address, routes to pods via labels

Namespace    → logical separation inside one cluster

The object hierarchy:
Deployment → ReplicaSet → Pod → Container

Service connects to pods independently, via matching labels — not through the Deployment.

---

## Tools & Environment
- **kubectl** — CLI to talk to any cluster
- **Docker Desktop Kubernetes (kind)** — local cluster I use day to day
- **minikube** — alternative local cluster
- **kops** — AWS cluster provisioning (explored, optional)

---

## Quick kubectl Reference

| Command | What it does |
|---|---|
| `kubectl get pods` | list pods |
| `kubectl get all` | list all resources in current namespace |
| `kubectl apply -f file.yaml` | create or update from yaml |
| `kubectl describe <type> <name>` | detailed info + events |
| `kubectl logs <pod>` | view container logs |
| `kubectl get pods -A` | all pods across all namespaces |
| `kubectl port-forward service/<name> 8080:80` | access a service locally |
| `kubectl rollout undo deployment/<name>` | rollback a deployment |
