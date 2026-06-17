# Pod Definitions

> Note: course tutorial uses an EC2 instance with a kops cluster.
> I used Docker Desktop's built-in Kubernetes (kind) locally instead.
> Pod yaml and kubectl commands are identical regardless of cluster backend.

---

## Goal
Learn raw Pod creation using a yaml definition file,
understand kubectl commands around pods, and learn the limitations
of editing raw Pods directly.

---

## Cluster Used
Docker Desktop → Kubernetes → kind cluster, 1 node
> Initially tried minikube, hit Hyper-V/VirtualBox driver conflicts.
> Enabled required Windows features (Virtual Machine Platform, Hyper-V),
> Docker Desktop started working, used its built-in Kubernetes instead.

---

## Files
- `vproapppod.yaml` — pod definition, originally tomcat-based example, switched to nginx

---

## Steps

### 1. Organize definitions folder
```bash
mkdir definitions
cd definitions
mkdir pod
cd pod
```

### 2. Write the pod definition
```bash
vim vproapppod.yaml
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: vproapp
  labels:
    app: vproapp
spec:
  containers:
    - name: appcontainer
      image: nginx:1.25
      ports:
        - name: vproapp-port
          containerPort: 80
```

### 3. Validate before creating
```bash
kubectl apply -f vproapppod.yaml --dry-run=client
```
> catches yaml syntax errors before touching the cluster

### 4. Create the pod
```bash
kubectl create -f vproapppod.yaml
# output: pod/vproapp created
```

### 5. Check pod status
```bash
kubectl get pod

# NAME       READY   STATUS              RESTARTS   AGE
# vproapp    0/1     ContainerCreating   0          4s
```

### 6. Investigate pod details
```bash
kubectl describe pod vproapp
# shows events, container status, image pull progress, IP, node
```

### 7. Edit and re-apply
```bash
# edited yaml image field from tomcat to nginx
kubectl apply -f vproapppod.yaml

# if "field is immutable" error appears:
kubectl delete -f vproapppod.yaml
kubectl apply -f vproapppod.yaml
```

---

## What I Learned
- `apiVersion` is the correct field name, not `version`
- YAML indentation must be typed manually — vim's autoindent only repeats
  the previous line's indentation, it doesn't add correct nesting
- YAML never allows tabs for indentation, only spaces
- `app: vproapp` label is just a naming convention, unrelated to which
  image/technology is running inside the container
- only `image` and `containerPort` actually depend on which software runs
  inside the container — `name`, `labels` are just organizational
- `kubectl create -f` fails if the resource already exists
- `kubectl apply -f` updates in place, but some Pod fields are immutable
  and require delete + recreate instead
- this immutability is exactly why Deployments exist — they handle
  rolling updates automatically instead of manual delete/recreate
- `kubectl apply --dry-run=client` validates yaml without touching the cluster
- cluster backend (minikube vs Docker Desktop k8s vs kops) doesn't matter
  for pod yaml or kubectl commands — they behave identically

---

## Gotchas & Mistakes
- typed `version: v1` instead of `apiVersion: v1` → invalid field, pod failed
- wrote yaml with zero indentation → broken structure, had to retype properly
- minikube failed with VirtualBox due to Hyper-V conflict
  fix: enabled Virtual Machine Platform + Hyper-V Windows features,
  switched to Docker Desktop's built-in Kubernetes instead
- Docker Desktop initially failed with "Virtualization support not detected"
  fix: turned out virtualization was fine, issue was Windows Hyper-V
  features being disabled, not BIOS
- edited image field after pod was created, `apply` behaved unexpectedly
  fix: delete + apply when fields are immutable
- assumed `app:` label needed to match the image/technology name
  clarified: label values are just naming convention, not tied to image

---

## Cheatsheet

| Command | What it does |
|---|---|
| `kubectl apply -f file.yaml --dry-run=client` | validate yaml without applying |
| `kubectl create -f file.yaml` | create resource, fails if exists |
| `kubectl apply -f file.yaml` | create or update, safe to repeat |
| `kubectl get pod` | list pods and status |
| `kubectl describe pod <name>` | detailed info, events, troubleshooting |
| `kubectl logs <name>` | view container logs |
| `kubectl delete pod <name>` | delete the pod |
| `kubectl delete -f file.yaml` | delete using the yaml definition |

---

## Next Section
→ [namespaces](../03-namespaces/README.md)
