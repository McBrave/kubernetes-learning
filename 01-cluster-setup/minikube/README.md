# Minikube — Local Kubernetes Cluster

> This is a local single-node cluster for learning and testing.
> Not for production use.

---

## Goal
Get a Kubernetes cluster running locally on your machine
so you can practice kubectl and K8s concepts without needing cloud infrastructure.

---

## How It Works
[Your Machine]

│

[Minikube]          ← runs a VM or container

│

[Single Node]        ← acts as both control plane + worker

│

[kubectl]            ← your CLI to talk to the cluster

Minikube simulates a real cluster on your laptop — same concepts,
same kubectl commands, just everything on one node.

---

## Prerequisites
- Docker installed and running
- kubectl installed
- 2 CPUs, 2GB RAM free minimum
- Linux / macOS / Windows (WSL2 recommended on Windows)

---

## Steps

### 1. Install minikube and kubectl

#### Windows (Chocolatey) — recommended if on Windows
```bash
choco install minikube kubernetes-cli -y

# kubernetes-cli   → this is kubectl
# -y               → auto confirm, no prompts
# installs both minikube AND kubectl in one command
```

#### Linux / macOS (curl)
```bash
# install kubectl
curl -LO "https://dl.k8s.io/release/$(curl -L -s \
  https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"

chmod +x kubectl
sudo mv kubectl /usr/local/bin/

# install minikube
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube
```

#### Verify both installed (all platforms)
```bash
minikube version
kubectl version --client
```

### 3. Start the cluster
```bash
minikube start --driver=docker

# --driver=docker  → runs cluster inside a Docker container
# first run takes 2-3 min, downloads the node image
```

Expected output:
✓ minikube v1.x.x on Linux

✓ Using the docker driver

✓ Starting control plane node minikube

✓ Done! kubectl is now configured to use "minikube"

### 4. Verify cluster is running
```bash
kubectl get nodes

# expected output:
# NAME       STATUS   ROLES           AGE   VERSION
# minikube   Ready    control-plane   1m    v1.x.x
```

STATUS must be `Ready` — if it says `NotReady` wait 30 seconds and try again.

### 5. Explore the cluster
```bash
# see all running system pods
kubectl get pods -A

# open the kubernetes dashboard in browser
minikube dashboard

# check cluster info
kubectl cluster-info
```

### 6. Stop and start again
```bash
# stop the cluster (saves state)
minikube stop

# start it back up
minikube start

# delete the cluster entirely (fresh start)
minikube delete
```

---

## What I Learned
- Minikube creates a real kubeconfig at `~/.kube/config` — kubectl reads this
  to know which cluster to talk to
- `minikube start` also automatically sets the kubectl context to minikube
- A context in kubectl = cluster + user + namespace combined
- The control plane and worker live on the same node in minikube —
  in real clusters they are separate machines

---

## Gotchas & Mistakes
- Started minikube before Docker was running → got driver error
  fix: start Docker first, then `minikube start`
- Ran kubectl commands and got "connection refused" →
  minikube was stopped, fix: `minikube start`
- `kubectl get nodes` showed NotReady → cluster was still initializing,
  just needed to wait 30 seconds
- Used `minikube delete` by mistake → lost everything,
  had to set up again (not a big deal for learning, just be aware)

---

## Cheatsheet

| Command | What it does |
|---|---|
| `minikube start` | start the cluster |
| `minikube stop` | stop but keep state |
| `minikube delete` | wipe everything |
| `minikube status` | check if running |
| `minikube dashboard` | open web UI in browser |
| `minikube ssh` | SSH into the node |
| `minikube logs` | see minikube logs if something breaks |
| `kubectl get nodes` | verify cluster is Ready |
| `kubectl get pods -A` | see all pods in all namespaces |
| `kubectl cluster-info` | show control plane address |
| `kubectl config get-contexts` | list all clusters kubectl knows about |
| `kubectl config use-context minikube` | switch to minikube if wrong context |

---

## Next Section
→ [kops — Production-like cluster on AWS](../kops/README.md)
