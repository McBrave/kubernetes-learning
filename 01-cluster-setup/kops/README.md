# Kops — Kubernetes Cluster on AWS

> Self-managed production-like Kubernetes cluster on AWS using kops.
> Unlike minikube, this creates real cloud infrastructure.

---

## Goal
Provision a real Kubernetes cluster on AWS using kops,
understanding every piece of infrastructure it creates.

---

## How It Works
[Your Machine / EC2]

│

[kops CLI]              ← reads config, talks to AWS

│

├── [S3 Bucket]           ← stores cluster state

├── [Route53]             ← DNS for cluster

└── [EC2 Instances]       ← control plane + worker nodes

---

## Prerequisites Overview
Domain          → needed for Route53 DNS

AWS Account     → IAM user, S3 bucket, Route53 hosted zone

EC2 Instance    → where you run kops and kubectl from

SSH Keys        → to access cluster nodes

kops            → cluster provisioning tool

kubectl         → CLI to talk to the cluster

AWS CLI         → to talk to AWS from terminal

---

## Part 1 — Domain Setup

### What and Why
kops uses DNS to identify cluster resources.
You need a domain so Route53 can create records like:
api.k8s.yourdomain.com        ← control plane endpoint

etcd.k8s.yourdomain.com       ← internal cluster comms

### Steps
AWS Console → Route53 → Hosted Zones → Create Hosted Zone

→ enter your domain name

→ type: Public hosted zone      ← must be public, not private

public = reachable from internet

private = only inside a VPC

→ create

→ copy the 4 NS records Route53 gives you

→ go to your domain registrar

→ update nameservers to those 4 NS records

> A public hosted zone is required because kops nodes need to
> resolve each other's DNS names over the internet.
> Private hosted zones only work inside a VPC and won't work here.

---

## Part 2 — AWS Account Setup

### 2a. IAM User

#### What and Why
Never use your AWS root account for kops.
Create a dedicated IAM user for kops to use.

#### Steps
AWS Console → IAM → Users → Create User

→ username: kops

→ attach policies directly

→ search and select: AdministratorAccess    ← full access for practice purposes

→ create user

→ go to the user → Security credentials

→ create access key → CLI type

→ save Access Key ID and Secret Access Key somewhere safe

> AdministratorAccess gives full AWS account access.
> This is acceptable for learning and practice only.
> In production always use least privilege —
> only grant the specific permissions kops actually needs.

---

### 2b. S3 Bucket

#### What and Why
kops stores the entire cluster state (config, keys, metadata)
in an S3 bucket — this is called the state store.
If you delete this bucket, kops loses track of your cluster.

```bash
# create the bucket
aws s3api create-bucket \
  --bucket kops-state-yourdomain \
  --region us-east-1 \
  --create-bucket-configuration LocationConstraint=us-east-1

# enable versioning (recommended — lets you roll back state)
aws s3api put-bucket-versioning \
  --bucket kops-state-yourdomain \
  --versioning-configuration Status=Enabled
```

> kops-state-yourdomain is just an example name.
> Replace it with any unique name you choose.
> S3 bucket names must be globally unique across all AWS accounts.
> Examples: kops-state-myproject, kops-state-learning, kops-state-john

---

## Part 3 — EC2 Instance (where you run kops from)

### What and Why
You need a machine to run kops, kubectl, and AWS CLI from.
Two options — your local machine or an EC2 instance.

Running kops from EC2 inside AWS has advantages:
- no need to configure AWS CLI credentials manually
  → attach an IAM role to EC2, it authenticates automatically
- faster AWS API calls since you're already inside AWS network
- no local environment setup issues

---

### Option A — EC2 Instance (recommended)

#### Launch the EC2
AWS Console → EC2 → Launch Instance

→ name: kops-management

→ OS: Ubuntu

→ instance type: t3.micro (enough just for running commands)

→ key pair: create new or use existing SSH key

→ security group: create new

Inbound rules:

Type: SSH

Port: 22

Source: My IP     ← your IP only, not 0.0.0.0/0

this means only YOU can SSH into this machine

never open SSH to the whole internet

→ launch

> Only one inbound rule needed — SSH from your IP.
> No need to open HTTP, HTTPS, or any other port
> since this EC2 is just a management machine, not serving traffic.

#### SSH into the EC2
```bash
ssh -i ~/.ssh/your-key.pem ubuntu@<ec2-public-ip>
```

#### Install tools on the EC2
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

#### Configure AWS CLI with IAM user credentials
```bash
aws configure

# it will ask for:
# AWS Access Key ID     → paste from IAM user you created in Part 2
# AWS Secret Access Key → paste from IAM user
# Default region        → us-east-1
# Default output format → json

# verify it works
aws sts get-caller-identity
# expected: returns your kops IAM user account details
```

---

### Option B — Local Machine (Windows)
```bash
choco install minikube kubernetes-cli -y
# then run aws configure same as above
```

---

## Part 4 — SSH Keys

### What and Why
kops needs SSH keys to access the EC2 nodes it creates during cluster setup.
You generate a key pair here — kops puts the public key on every
cluster node via the --ssh-public-key flag during cluster creation.
Private key (id_ed25519)     → stays on your machine, like your house key

Public key (id_ed25519.pub)  → kops puts this on every cluster node

like the lock on the door

When you SSH into a node later, your private key is checked
against the public key on the node — if they match, access is granted.

### Generate the key pair
```bash
ssh-keygen -t ed25519

# accept default location (~/.ssh/id_ed25519)
# add a passphrase for extra security or leave empty

# this creates two files:
# ~/.ssh/id_ed25519        ← private key, never share this
# ~/.ssh/id_ed25519.pub    ← public key, this goes to cluster nodes
```

### Verify keys exist
```bash
ls ~/.ssh/
# you should see both id_ed25519 and id_ed25519.pub
```

> ed25519 is the modern SSH key type — faster and more secure than
> the older rsa type. Always prefer ed25519 for new keys.

---

## Part 5 — Create and Launch the Cluster

### 5a. Set Environment Variables
```bash
# set for current session
export KOPS_STATE_STORE=s3://kops-state-yourdomain
export NAME=mydomain

# make both permanent so they survive terminal restarts
echo "export KOPS_STATE_STORE=s3://kops-state-yourdomain" >> ~/.bashrc
echo "export NAME=mydomain" >> ~/.bashrc

# apply immediately without restarting terminal
source ~/.bashrc

# verify both are set
echo $KOPS_STATE_STORE
echo $NAME
# expected:
# s3://kops-state-yourdomain
# mydomain
```

> source ~/.bashrc reloads the file immediately.
> Without it you would have to close and reopen the terminal
> for the variables to take effect.

---

### 5b. Create Cluster Config
This writes the cluster config to S3 — does NOT create anything on AWS yet:

```bash
kops create cluster \
  --name=$NAME \
  --state=$KOPS_STATE_STORE \
  --zones=us-east-1a,us-east-1b \
  --node-count=2 \
  --node-size=t3.small \
  --control-plane-size=t3.medium \
  --dns-zone=$NAME \
  --node-volume-size=12 \
  --control-plane-volume-size=12 \
  --ssh-public-key ~/.ssh/id_ed25519.pub
```

#### What each flag does
--name                      → cluster name, matches your Route53 domain

--state                     → which S3 bucket to store cluster state in

--zones                     → AWS availability zones to spread across

two zones = high availability

--node-count                → how many worker nodes to create

--node-size                 → EC2 instance type for worker nodes

--control-plane-size        → EC2 instance type for control plane

larger because it runs etcd,

API server and scheduler

--dns-zone                  → Route53 hosted zone for cluster DNS

--node-volume-size          → EBS disk size in GB for worker nodes

--control-plane-volume-size → EBS disk size in GB for control plane

--ssh-public-key            → public key to put on every node

~/.ssh/id_ed25519.pub was generated in Part 4

---

### 5c. Dry Run — Review Before Applying
```bash
# preview what kops WOULD create — nothing is built yet
kops update cluster --name=$NAME

# review the output carefully
# if everything looks correct proceed to next step
```

> always dry run first — without --yes kops only shows a plan

---

### 5d. Apply and Launch the Cluster
```bash
kops update cluster \
  --name=$NAME \
  --yes \
  --admin
```

#### What each flag does
--yes     → actually apply changes and create everything on AWS

without it, just shows a dry run plan

--admin   → writes kubectl credentials to ~/.kube/config automatically

so kubectl works immediately after cluster comes up

---

### 5e. Wait for Cluster to Come Up
```bash
# wait up to 10 minutes for cluster to be ready
kops validate cluster --wait 10m

# once valid, check nodes
kubectl get nodes

# expected:
# NAME              STATUS   ROLES
# i-xxxxx.ec2      Ready    control-plane
# i-xxxxx.ec2      Ready    node
# i-xxxxx.ec2      Ready    node
```

---

## What I Learned
- kops create cluster only writes config to S3 — nothing on AWS until update --yes
- --yes flag is the real trigger — always dry run first without it
- --admin flag saves manually copying kubeconfig after cluster comes up
- two zones = high availability, nodes spread across both
- t3.medium for control plane because it runs etcd, API server, scheduler
- id_ed25519 is modern SSH key type, more secure and faster than rsa
- running kops from EC2 with IAM role skips aws configure entirely
- Route53 hosted zone must be public — private zones only work inside a VPC
- AdministratorAccess is fine for learning but never use in production
- S3 bucket name just needs to be globally unique — domain name not required

---

## Gotchas & Mistakes
- used root account credentials in AWS CLI → bad practice
  fix: always create a separate IAM user for tools like kops

- used specific IAM policies → kept hitting AccessDeniedException errors
  fix: for learning attach AdministratorAccess and focus on kops, not IAM debugging
  remember: in production always use least privilege

- forgot to set NAME variable → kops commands kept asking for cluster name
  fix: export NAME=mydomain and add to ~/.bashrc

- forgot source ~/.bashrc after adding variables → variables not found
  fix: always run source ~/.bashrc after editing it

- ran kops update --yes without dry run first → unexpected changes
  fix: always run kops update cluster without --yes first to review

- forgot --admin flag on update → kubectl had no config
  fix: kops export kubecfg --admin

- Route53 hosted zone set to private → cluster DNS failed
  fix: must use public hosted zone for kops

- cluster took longer than expected to be Ready → DNS not propagated yet
  fix: verify with dig NS mydomain before creating cluster

- opened SSH in security group to 0.0.0.0/0 → security risk
  fix: always restrict SSH inbound rule to your IP only

- S3 versioning not enabled from the start → couldn't roll back state
  fix: always enable versioning right after creating the bucket

---

## Cheatsheet

| Command | What it does |
|---|---|
| `export NAME=mydomain` | set cluster name variable |
| `export KOPS_STATE_STORE=s3://...` | set state store variable |
| `echo "export ..." >> ~/.bashrc` | make variable permanent |
| `source ~/.bashrc` | apply bashrc changes immediately |
| `echo $NAME` | verify variable is set |
| `aws sts get-caller-identity` | verify which AWS user is active |
| `aws s3 ls` | list your S3 buckets |
| `aws route53 list-hosted-zones` | list Route53 zones |
| `kops create cluster ...` | write cluster config to S3 only |
| `kops update cluster --name=$NAME` | dry run, preview changes |
| `kops update cluster --name=$NAME --yes --admin` | create everything on AWS |
| `kops validate cluster --wait 10m` | wait for cluster to be ready |
| `kops get cluster` | list clusters kops knows about |
| `kops delete cluster --name=$NAME --yes` | destroy everything |
| `kops export kubecfg --admin` | fix missing kubeconfig |
| `kubectl get nodes` | verify nodes are Ready |
| `dig NS mydomain` | verify DNS is propagating |

---

## Next Section
→ [02 — Deployments](../../02-deployments/README.md)
