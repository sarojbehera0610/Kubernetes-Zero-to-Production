# Kubernetes Networking

## Intra-Pod Communication

Containers inside the same Pod share the same network namespace.
They communicate using `localhost`.
Container A (nginx:80) ←→ localhost:6379 ←→ Container B (redis:6379)
Same IP address, same Pod

### Multi-container Pod Manifest
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: multi-container-pod
spec:
  containers:
    - name: main-container
      image: nginx
      ports:
        - containerPort: 80
    - name: sidecar-container
      image: redis
      ports:
        - containerPort: 6379
```

---

## Inter-Pod Communication

Each Pod gets a unique IP. Pods communicate directly without NAT
(flat network model).
Pod A (192.168.1.10) → Pod B (192.168.1.20):8080

Or via a Service for stable endpoint:
Pod A → Service ClusterIP → kube-proxy → Pod B

---

## Hands-on: NodePort and LoadBalancer

Verified on EKS cluster (ap-south-1):
- NodePort: accessed via `<NodeIP>:30001`
- LoadBalancer: AWS ELB created automatically, accessed via external DNS

## Key Learning
- Intra-pod = localhost, same IP
- Inter-pod = Pod IP or Service
- Kubernetes guarantees all pods can reach all other pods without NAT