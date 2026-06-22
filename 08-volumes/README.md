# 08 — Volumes

## Goal
Persist data beyond a container's life and share data between containers in a Pod. By default a container's filesystem is ephemeral: when a container restarts (crash, reschedule, rolling update), everything written inside it is wiped. Volumes store data **outside** the container so it survives.

## How It Works
A **Volume** is a directory defined at the **Pod** level and mounted into a container at a path you choose. Two YAML parts always work together:
- `spec.volumes` — *defines* the storage (you own a hard drive)
- `spec.containers.volumeMounts` — *attaches* it into a container at a path (you plug the drive into a specific port)

```
        POD
┌───────────────────────────┐
│   Container                │
│   ┌─────────────────┐      │
│   │ mountPath:      │◄─────┼──┐  mounted into container
│   │ /usr/share/data │      │  │
│   └─────────────────┘      │  │
│   ┌─────────────────┐      │  │
│   │ Volume "my-data"│◄─────┼──┘  defined at pod level
│   └─────────────────┘      │
└───────────────────────────┘
```

### Volume types (early ones)

**`emptyDir`** — scratch space, lives as long as the Pod lives.
```
Pod created                  ──> volume created (empty)
Container crashes & restarts ──> data SURVIVES ✓ (pod still alive)
Pod deleted                  ──> data GONE ✗
```
Use for temporary cache, or **sharing files between two containers in the same Pod**.

**`hostPath`** — mounts a directory from the *node's* filesystem into the Pod.
```
   POD                      NODE (the machine)
┌──────────┐              ┌────────────────────┐
│Container │              │                    │
│/data ◄───┼──────────────┼── /mnt/host-data   │
└──────────┘              └────────────────────┘
        survives pod deletion (tied to the node)
```
Fine for local testing on kind/Docker Desktop. **Avoid in real multi-node clusters** — it pins the Pod to one node and is a security/portability risk.

### The real pattern: PV + PVC
Decouples the app developer ("I need 5Gi") from the storage details (which disk, which cloud).
```
┌─────────────┐  "I need 5Gi"  ┌──────────────┐   binds to   ┌──────────────┐
│ Pod         │ ─────────────► │ PVC          │ ───────────► │ PV           │
│ uses a PVC  │                │ (the REQUEST)│              │ (real disk)  │
└─────────────┘                └──────────────┘              └──────────────┘
  developer's view                the "claim"                  admin's view
```
- **PV (PersistentVolume)** = an actual piece of storage in the cluster.
- **PVC (PersistentVolumeClaim)** = a *request* for storage. The Pod references the **PVC**, not the PV.
- Kubernetes **binds** a matching PV to the PVC.

```
ANALOGY:
PV  = a parking spot that exists in the garage
PVC = your reservation ticket ("I need one spot")
Pod = your car — it parks based on the ticket; you never pick the exact spot
```

### StorageClass — dynamic provisioning
Creating PVs by hand is tedious. A **StorageClass** auto-creates a matching PV when you create a PVC.
```
WITHOUT StorageClass:  manually create PV  ──>  PVC binds to it
WITH StorageClass:     create PVC only     ──>  PV auto-created on demand ✓
```
On kind/Docker Desktop there's a default StorageClass (`standard`) backed by local storage, so a bare PVC usually just works.

### Fits the mental model
```
Deployment → ReplicaSet → Pod → Container
                            │
                            ├── volumeMounts (container's view: a path)
                            └── volumes (pod's view: the storage)
                                   │
                                   └── persistentVolumeClaim → PV → real disk
```

## Files
- `emptydir-pod.yaml` — Pod with an `emptyDir` scratch volume
- `hostpath-pod.yaml` — Pod mounting a node directory via `hostPath`
- `pv.yaml` — a PersistentVolume (hostPath-backed for local testing)
- `pvc.yaml` — a PersistentVolumeClaim requesting storage
- `pvc-pod.yaml` — Pod consuming the PVC

### `emptydir-pod.yaml`
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: emptydir-demo
spec:
  containers:
    - name: writer
      image: busybox
      command: ["sh", "-c", "echo hello > /cache/data.txt && sleep 3600"]
      volumeMounts:
        - name: shared-cache      # must match the volume name below
          mountPath: /cache       # where it appears inside THIS container
  volumes:
    - name: shared-cache          # the volume definition
      emptyDir: {}                # type: empty directory
```

### `hostpath-pod.yaml`
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: hostpath-demo
spec:
  containers:
    - name: app
      image: nginx
      volumeMounts:
        - name: host-storage
          mountPath: /usr/share/nginx/html
  volumes:
    - name: host-storage
      hostPath:
        path: /mnt/data           # path on the node
        type: DirectoryOrCreate
```

### `pv.yaml`
```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: my-pv
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteOnce
  hostPath:                       # backing with hostPath for local testing
    path: /mnt/data
```

### `pvc.yaml`
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
```

### `pvc-pod.yaml`
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pvc-demo
spec:
  containers:
    - name: app
      image: nginx
      volumeMounts:
        - name: my-storage
          mountPath: /usr/share/nginx/html
  volumes:
    - name: my-storage
      persistentVolumeClaim:
        claimName: my-pvc         # Pod points to the CLAIM, not the PV
```

## Steps
```bash
# --- emptyDir: data survives container restart, not pod deletion ---
kubectl apply -f emptydir-pod.yaml
kubectl exec emptydir-demo -- cat /cache/data.txt      # hello

# --- PV + PVC flow ---
kubectl apply -f pv.yaml
kubectl apply -f pvc.yaml
kubectl get pv,pvc                                     # STATUS should be Bound
kubectl apply -f pvc-pod.yaml

# write a file through the mounted volume
kubectl exec pvc-demo -- sh -c 'echo "<h1>persisted</h1>" > /usr/share/nginx/html/index.html'

# delete the POD (not the PVC) and recreate — data should still be there
kubectl delete pod pvc-demo
kubectl apply -f pvc-pod.yaml
kubectl exec pvc-demo -- cat /usr/share/nginx/html/index.html   # still there ✓

# inspect / clean up
kubectl describe pvc my-pvc
kubectl delete -f pvc-pod.yaml -f pvc.yaml -f pv.yaml
```

## What I Learned
- Container filesystems are **ephemeral**; volumes store data outside the container so it survives restarts.
- A volume is defined at the **Pod** level (`spec.volumes`) and attached into a container with `volumeMounts`; the `name` links the two.
- `emptyDir` is tied to the Pod's life — survives container restarts, dies with the Pod.
- `hostPath` mounts a node directory; great for local testing, bad for real multi-node clusters.
- **PVC/PV decouples** the app's storage *request* from the actual disk, so the Pod YAML stays the same across clouds and local. The Pod always references the **PVC**, never the PV directly.
- A **StorageClass** enables dynamic provisioning — create a PVC and a PV is made automatically.
- `accessModes`: `ReadWriteOnce` (one node RW — most common), `ReadOnlyMany`, `ReadWriteMany` (needs NFS-like storage).

## Gotchas & Mistakes
- `volumeMounts[].name` **must exactly match** a `volumes[].name`, or the Pod won't start.
- `emptyDir` data is **lost when the Pod is deleted** — it is *not* persistent storage, despite surviving container restarts. Don't use it for databases.
- The Pod points to the **PVC**, not the PV. Pointing at the PV directly is a common beginner mistake.
- A PVC stuck in `Pending` usually means no PV matches its size/`accessModes`, or no default StorageClass exists.
- `ReadWriteMany` does **not** work with `hostPath` or single-node local disks — needs networked storage (NFS, etc.).
- `hostPath` paths are relative to the **node**, not your laptop — on kind the "node" is a container, so the path lives inside that container.

## Cheatsheet
```bash
kubectl get pv                       # list persistent volumes
kubectl get pvc                      # list claims (watch for Bound/Pending)
kubectl get storageclass             # list storage classes (sc)
kubectl describe pvc <name>          # why a claim is Pending
kubectl describe pv <name>           # see what a PV is bound to
kubectl exec <pod> -- ls /mountpath  # verify the mount inside the container
kubectl delete pvc <name>            # release a claim
```
```
volumes:                             # POD level — defines storage
  - name: data
    emptyDir: {}                     #  OR hostPath: {path: ...}
    #                                #  OR persistentVolumeClaim: {claimName: ...}

volumeMounts:                        # CONTAINER level — attaches it
  - name: data                       #  must match a volume name
    mountPath: /path/in/container
```

## Next Section
➡️ [09 — ConfigMaps & Secrets](../09-configmaps-and-secrets/README.md)
