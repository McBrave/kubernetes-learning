# Kops — Kubernetes Cluster on AWS

> Self-managed production-like Kubernetes cluster on AWS using kops.
> Unlike minikube, this creates real cloud infrastructure.

---

## Goal
Provision a real Kubernetes cluster on AWS using kops,
understanding every piece of infrastructure it creates.

---

## How It Works
[Your Machine]

│

[kops CLI]          ← reads config, talks to AWS

│

├── [S3 Bucket]        ← stores cluster state

├── [Route53]          ← DNS for cluster

└── [EC2 Instances]    ← control plane + worker nodes

---

## Prerequisites Overview
Domain          → needed for Route53 DNS

VM / Machine    → where you run kops and kubectl

kops            → the cluster provisioning tool

kubectl         → CLI to talk to the cluster

SSH keys        → to access cluster nodes

AWS CLI         → to talk to AWS from terminal

AWS Account     → with IAM user, S3 bucket, Route53

---

## Part 1 — Domain Setup

### What and Why
kops uses DNS to identify cluster resources.
You need a domain so Route53 can create records like:
api.k8s.yourdomain.com        ← control plane endpoint

etcd.k8s.yourdomain.com       ← internal cluster comms

### Steps
```bash
# if you already have a domain, point its nameservers to Route53
# Route53 → Hosted Zones → Create Hosted Zone → enter your domain
# copy the 4 NS records Route53 gives you
# go to your registrar → update nameservers to those 4
```

> If you don't have a domain, Route53 can register one
> or use a subdomain of an existing domain you own

---
## Part 2 — SSH Keys

### What and Why
kops uses SSH keys to access the EC2 nodes it creates.
You generate a key pair — kops puts the public key on every node.

```bash
# generate key pair
ssh-keygen -t rsa -b 4096 -f ~/.ssh/kops_rsa

# -t rsa          → key type
# -b 4096         → key strength (bits)
# -f              → where to save it

# this creates two files:
# ~/.ssh/kops_rsa        ← private key (never share this)
# ~/.ssh/kops_rsa.pub    ← public key (goes to EC2 nodes)
```

## Part 3 — Machine Setup (where you run kops from)

### What and Why
You need a machine to run kops, kubectl, and AWS CLI from.
Two options — local machine or an EC2 instance.

---

### Option A — Local Machine (Windows)

```bash
choco install minikube kubernetes-cli -y
```

---

### Option B — EC2 Instance (recommended for AWS workflows)

#### What and Why
Running kops from an EC2 instance inside AWS has advantages:
- no need to configure AWS CLI credentials manually
  → attach an IAM role to the EC2 instead, it authenticates automatically
- faster AWS API calls since you're already inside AWS network
- no local environment setup issues

#### Steps
AWS Console → EC2 → Launch Instance

→ choose Ubuntu or Amazon Linux 2

→ instance type: t3.micro (enough for running commands)

→ key pair: use your existing SSH key

→ IAM instance profile: attach the IAM role with kops permissions

→ launch

Then SSH into it:
```bash
ssh -i ~/.ssh/your-key.pem ubuntu@<ec2-public-ip>
```

Then install tools on the EC2:
```bash
# kubectl
curl -LO "https://dl.k8s.io/release/$(curl -L -s \
  https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl && sudo mv kubectl /usr/local/bin/

# kops
curl -Lo kops https://github.com/kubernetes/kops/releases/download/$(curl -s \
  https://api.github.com/repos/kubernetes/kops/releases/latest \
  | grep tag_name | cut -d '"' -f 4)/kops-linux-amd64
chmod +x kops && sudo mv kops /usr/local/bin/

# aws cli
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip && sudo ./aws/install
```

#### Verify installs
```bash
kops version
kubectl version --client
aws --version
```

> Key benefit: because the EC2 has an IAM role attached,
> you skip `aws configure` entirely — AWS CLI picks up
> credentials automatically from the instance metadata

---

---

## Part 4 — AWS Account Setup

### 4a. IAM User

#### What and Why
Never use your AWS root account for kops.
Create a dedicated IAM user with only the permissions kops needs.

#### Required Permissions
AmazonEC2FullAccess

AmazonRoute53FullAccess

AmazonS3FullAccess

IAMFullAccess

AmazonVPCFullAccess

#### Steps
AWS Console → IAM → Users → Create User

→ attach policies above

→ create access key (CLI type)

→ save Access Key ID and Secret Access Key

#### Configure AWS CLI with the IAM user
```bash
aws configure

# it will ask for:
# AWS Access Key ID     → paste from IAM user
# AWS Secret Access Key → paste from IAM user
# Default region        → eu-central-1 (or your region)
# Default output format → json

# verify it works
aws sts get-caller-identity
# expected: returns your IAM user account details
```

---

### 4b. S3 Bucket

#### What and Why
kops stores the entire cluster state (config, keys, metadata)
in an S3 bucket — this is called the state store.
If you delete this bucket, kops loses track of your cluster.

```bash
# create the bucket
aws s3api create-bucket \
  --bucket kops-state-yourdomain \
  --region eu-central-1 \
  --create-bucket-configuration LocationConstraint=eu-central-1

# enable versioning (recommended — lets you roll back state)
aws s3api put-bucket-versioning \
  --bucket kops-state-yourdomain \
  --versioning-configuration Status=Enabled

# set as environment variable so kops finds it automatically
export KOPS_STATE_STORE=s3://kops-state-yourdomain
```

> Add the export line to ~/.bashrc or ~/.zshrc
> so you don't have to set it every session

---

### 4c. Route53

#### What and Why
kops uses Route53 to create DNS records for the cluster.
The hosted zone must match the domain you're using for the cluster.

```bash
# create hosted zone
aws route53 create-hosted-zone \
  --name k8s.yourdomain.com \
  --caller-reference $(date +%s)

# list hosted zones to verify
aws route53 list-hosted-zones

# copy the 4 NS records and add them to your domain registrar
# under a subdomain record if using subdomain delegation
## Part 5 — Create and Launch the Cluster

### 5a. Set Environment Variable
```bash
export KOPS_STATE_STORE=s3://kopsstate956

# add to ~/.bashrc so it persists
echo "export KOPS_STATE_STORE=s3://kopsstate956" >> ~/.bashrc
```

### 5b. Create Cluster Config
This writes the cluster config to S3 — does NOT create anything on AWS yet:

```bash
kops create cluster \
  --name=mydomain \
  --state=s3://kopsstate956 \
  --zones=us-east-1a,us-east-1b \
  --node-count=2 \
  --node-size=t3.small \
  --control-plane-size=t3.medium \
  --dns-zone=mydomain \
  --node-volume-size=12 \
  --control-plane-volume-size=12 \
  --ssh-public-key ~/.ssh/id_ed25519.pub
```

#### What each flag does
--name                  → cluster name, matches your Route53 domain

--state                 → which S3 bucket to store cluster state in

--zones                 → which AWS availability zones to spread across

two zones = high availability

--node-count            → how many worker nodes to create

--node-size             → EC2 instance type for worker nodes

--control-plane-size    → EC2 instance type for control plane

larger than nodes because it runs more components

--dns-zone              → Route53 hosted zone for cluster DNS

--node-volume-size      → EBS disk size in GB for worker nodes

--control-plane-volume-size → EBS disk size in GB for control plane

--ssh-public-key        → public key to put on every node

id_ed25519 is newer and more secure than rsa

### 5c. Apply and Launch the Cluster
This is what actually creates everything on AWS:

```bash
kops update cluster \
  --name=mydomain \
  --state=s3://kopsstate956 \
  --yes \
  --admin
```

#### What each flag does
--yes     → actually apply changes (without it, just shows a dry run plan)

--admin   → writes kubectl credentials to ~/.kube/config automatically

so kubectl works immediately after cluster comes up

> Without --yes the command just previews what would be created.
> Always run without --yes first to review, then add --yes to apply.

### 5d. Wait for Cluster to Come Up
```bash
# watch nodes come up (takes 5-10 min)
kops validate cluster --wait 10m

# once valid, check nodes
kubectl get nodes

# expected:
# NAME                STATUS   ROLES           
# i-xxxxx.ec2        Ready    control-plane   
# i-xxxxx.ec2        Ready    node            
# i-xxxxx.ec2        Ready    node            
```

---

## What I Learned
- kops create cluster only writes config to S3 — nothing on AWS until update --yes
- --yes flag is the real trigger — always dry run first without it
- --admin flag is convenient — saves manually copying kubeconfig
- two zones (us-east-1a, us-east-1b) = high availability, nodes spread across both
- t3.medium for control plane because it runs etcd, API server, scheduler
- id_ed25519 is a newer SSH key type, more secure than rsa
- running kops from EC2 with IAM role skips credentials setup entirely
- node-volume-size 12GB is minimum for learning — increase for real workloads

---

## Gotchas & Mistakes
- ran kops update --yes before checking the plan → always dry run first
- forgot --admin flag → kubectl had no config, had to export kubeconfig manually
  fix: kops export kubecfg --admin
- cluster took longer than expected to be Ready → Route53 DNS not fully propagated
  fix: dig NS mydomain to verify DNS before creating cluster
- used t3.micro for control plane → ran out of memory, scheduler kept crashing
  fix: t3.medium minimum for control plane
- node-volume too small → pods got evicted due to disk pressure
  fix: 12GB minimum, 20GB recommended for real use

---

## Cheatsheet

| Command | What it does |
|---|---|
| `kops create cluster ...` | write cluster config to S3 (no AWS resources yet) |
| `kops update cluster --yes` | actually create everything on AWS |
| `kops validate cluster` | check if cluster is healthy |
| `kops validate cluster --wait 10m` | wait up to 10 min for cluster to be ready |
| `kops get cluster` | list clusters kops knows about |
| `kops delete cluster --yes` | destroy everything (EC2, DNS, volumes) |
| `kops export kubecfg --admin` | manually write kubeconfig if you forgot --admin |
| `kubectl get nodes` | verify nodes are Ready |

---

## Next Section
→ [Deployments](../../02-deployments/README.md)
