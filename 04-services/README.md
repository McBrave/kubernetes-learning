# Services

> A Service gives pods a stable network address since pod IPs
> change constantly as they restart.

---

## Goal
Understand the three Service types — ClusterIP, NodePort, LoadBalancer —
and how to access services on a local kind cluster.

---

## Service Types

### ClusterIP (default)
[Pod A] → [Service: ClusterIP] → [Pod B]
Internal only. Used for pod-to-pod communication (e.g. app to database).

### NodePort
[User] → [Node IP : NodePort] → [Service] → [Pod]
Exposes app via a port (30000-32767) on every node.

### LoadBalancer
[Internet] → [Cloud Load Balancer] → [Service] → [Pod]
Requires a real cloud provider (AWS/GCP/Azure) to provision an actual
load balancer. On local clusters (kind/minikube) it stays `<pending>`.

---

## Files
- `clusterip-service.yaml`    — internal only, pod-to-pod
- `nodeport-service.yaml`     — exposes via node port, tested with port-forward
- `loadbalancer-service.yaml` — cloud-only type, stays pending locally

---

## Steps

### 1. Apply all three service types
```bash
kubectl apply -f clusterip-service.yaml
kubectl apply -f nodeport-service.yaml
kubectl apply -f loadbalancer-service.yaml
```

### 2. Compare them side by side
```bash
kubectl get svc

# NAME                     TYPE           CLUSTER-IP     EXTERNAL-IP   PORT(S)
# vproapp-clusterip        ClusterIP      10.0.0.5               80/TCP
# vproapp-nodeport         NodePort       10.0.0.6               80:30080/TCP
# vproapp-loadbalancer     LoadBalancer   10.0.0.7            80:31234/TCP
```

> notice LoadBalancer's EXTERNAL-IP stays `<pending>` forever on a local
> kind cluster — confirms it needs a real cloud provider to work

### 3. Test ClusterIP (only reachable from inside the cluster)
```bash
# exec into another pod and curl the clusterip service
kubectl run test-pod --image=busybox -it --rm -- sh
wget -O- vproapp-clusterip
```

### 4. Test NodePort (via port-forward since on kind)
```bash
kubectl port-forward service/vproapp-nodeport 8080:80
# visit http://localhost:8080
```

### 5. Clean up after testing
```bash
kubectl delete -f clusterip-service.yaml
kubectl delete -f nodeport-service.yaml
kubectl delete -f loadbalancer-service.yaml
```bash
# kind nodes are docker containers, node IP isn't directly reachable
# use port-forward instead, works regardless of cluster type

kubectl port-forward service/vproapp-service 8080:80

# now visit
http://localhost:8080
```

---

## What I Learned
- Service.selector matches Pod labels — that's how Service finds its pods
- three ports matter for NodePort: port, targetPort, nodePort
- ClusterIP is the default, internal-only
- LoadBalancer needs a real cloud provider — stays pending on local clusters
- on kind specifically, NodePort isn't directly reachable from a browser
  via node IP like it is on minikube — kubectl port-forward is the
  most reliable method for local access regardless of cluster type

---

## Gotchas & Mistakes
- tried minikube-style "get node ip and visit nodeIP:nodePort" approach,
  doesn't work the same way on kind since nodes run as Docker containers
  fix: used kubectl port-forward instead

---

## Cheatsheet

| Command | What it does |
|---|---|
| `kubectl get svc` | list services and their ports |
| `kubectl describe svc <name>` | detailed info, endpoints, events |
| `kubectl get endpoints <name>` | see which pods a service routes to |
| `kubectl port-forward service/<name> <local-port>:<service-port>` | access service from localhost |
| `kubectl get nodes -o wide` | see node IPs (not directly useful on kind) |

---

## Next Section
→ [Deployments](../05-deployments/README.md)
