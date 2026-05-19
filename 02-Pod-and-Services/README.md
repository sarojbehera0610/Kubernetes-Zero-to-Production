# Pods and Services

## Pod

Smallest deployable unit in Kubernetes. Wraps one or more containers
with shared network and storage.

### Pod Lifecycle
Pending → Running → Succeeded / Failed → Terminated

### Sample Pod Manifest
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
  labels:
    app: my-app
spec:
  containers:
    - name: my-container
      image: nginx:latest
      ports:
        - containerPort: 80
```

---

## Services

Stable networking layer on top of Pods. Pods die and restart with new IPs
— Services provide a consistent endpoint.

### Service Types

| Type | Access | Use Case |
|---|---|---|
| ClusterIP | Inside cluster only | Internal microservice communication |
| NodePort | NodeIP:NodePort | Dev/testing external access |
| LoadBalancer | External IP (AWS ELB) | Production external traffic |

### Combined Namespace + Pod + Service Manifest
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: my-ns
---
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
  namespace: my-ns
  labels:
    app: my-app
spec:
  containers:
    - name: nginx
      image: nginx:latest
      ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: my-svc
  namespace: my-ns
spec:
  type: NodePort
  selector:
    app: my-app
  ports:
    - port: 80
      targetPort: 80
      nodePort: 30001
```

## Key Learning
- Service selector must exactly match Pod labels
- NodePort range: 30000–32767
- LoadBalancer creates AWS ELB automatically on EKS