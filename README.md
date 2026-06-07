# Kubernetes Zero to Production 🚀

A hands-on learning repository documenting my Kubernetes journey from core concepts
to production-grade workloads on AWS EKS. Each folder contains topic-specific
manifest files, notes, and real outputs from a live EKS cluster on AWS (ap-south-1).

---

## 🧑‍💻 Environment

- **Cluster**: AWS EKS (`demo-ekscluster`) — ap-south-1
- **Nodes**: EC2 t2.micro (2 worker nodes)
- **Tools**: kubectl, eksctl, AWS CLI
- **Local**: Windows + VS Code + Git Bash

---

## 📁 Repository Structure

| Folder | Topic |
|---|---|
| 01-Architecture | Kubernetes architecture — control plane & worker node components |
| 02-Pod-and-Services | Pod lifecycle, Service types and manifest files |
| 03-Networking | Intra-pod and inter-pod communication |
| 04-Controllers | ReplicationController vs ReplicaSet |
| 05-Deployments | Deployment strategies — Recreate, Rolling Update |
| 06-StatefulSet | StatefulSet vs Deployment, Headless vs Normal Service DNS test |
| 07-DaemonSet | DaemonSet use cases, node-level pod scheduling |
| 08-ConfigMap-and-Secrets | Externalising config, managing sensitive data |
| 09-Volumes-PV-PVC | Persistent Volumes, PVCs, EBS CSI driver, dynamic provisioning |
| 10-HPA | Horizontal Pod Autoscaler — CPU-based autoscaling with load testing |
| 11-Ingress | NGINX Ingress Controller — path-based routing for frontend & backend |
| 12-Three-Tier-App | Full 3-tier app deployment — React + Spring Boot + MariaDB on EKS |

---

## 🗺️ Topics Covered

### Why Kubernetes?
- Container orchestration challenges — scalability, HA, load balancing
- Why Kubernetes over manual container management
- Open source, portable, extensible, self-healing

### Architecture
- **Control Plane**: API Server, etcd, Scheduler, Controller Manager, cloud-controller-manager
- **Worker Node**: kubelet, kube-proxy, container runtime
- **Pod**: smallest deployable unit

### Cluster Setup
- EKS cluster creation using `eksctl`
- `kubectl` installation and kubeconfig setup
- IAM instance profile for EC2 authentication

### Pods & Services
- Pod lifecycle: Pending → Running → Succeeded/Failed
- Service types: ClusterIP, NodePort, LoadBalancer
- Writing combined Namespace + Pod + Service manifests

### Networking
- **Intra-pod**: containers share same IP, communicate via localhost
- **Inter-pod**: unique Pod IPs, flat network model, no NAT
- NodePort and LoadBalancer (AWS ELB) hands-on

### Controllers
- ReplicationController (legacy) vs ReplicaSet (modern)
- Selector difference: equality-based vs set-based
- Hands-on manifest files for both

### Deployments
- Recreate strategy — all pods terminated then recreated
- Rolling Update — incremental pod replacement, zero downtime
- `kubectl rollout` commands — status, history, undo
- `kubectl set image` for triggering rollouts

### StatefulSet
- Stateful vs Stateless applications
- Ordered pod creation (pod-0 → pod-1 → pod-2)
- Stable pod DNS identity
- **Headless vs Normal Service DNS comparison** (hands-on nslookup test)
- PersistentVolumeClaims — volumeClaimTemplates

### DaemonSet
- Runs one pod per node — log collectors, monitoring agents, network plugins
- Node selector and tolerations
- Hands-on manifest with rolling update strategy

### ConfigMap & Secrets
- Decoupling config from container images
- ConfigMap — env vars, volume mounts, config files
- Secrets — base64-encoded sensitive data, secretKeyRef
- Practical examples with Spring Boot datasource config

### Volumes, PV & PVC
- Ephemeral vs Persistent storage
- PersistentVolume (PV) and PersistentVolumeClaim (PVC) lifecycle
- Static vs Dynamic provisioning
- **EBS CSI Driver** setup on EKS with IRSA/OIDC
- `volumeClaimTemplates` in StatefulSet vs shared PVC in Deployment
- `reclaimPolicy: Retain` — data survival after PVC deletion

### Horizontal Pod Autoscaler (HPA)
- HPA scaling formula — `desiredReplicas = ceil(current × actual/target)`
- Metrics Server installation on EKS
- CPU-based autoscaling — scale-out and cool-down behaviour
- Live load testing with BusyBox to trigger scale events
- **Troubleshooting**: `<unknown>` targets, missing resource requests, Pending pods

### Ingress
- Ingress vs Service LoadBalancer — cost and routing comparison
- NGINX Ingress Controller deployment on EKS
- **Path-based routing** — `/api/*` → backend, `/*` → frontend
- `rewrite-target` annotation with regex capture groups
- `ImplementationSpecific` pathType for regex rewrites
- Subnet tagging for NLB provisioning on EKS
- **Troubleshooting**: empty ADDRESS, 404 on paths, CORS, missing ingressClassName

### 3-Tier Application Deployment
- Full stack: React (frontend) + Spring Boot (backend) + MariaDB (database)
- Deployment order — DB → Backend → Frontend (dependency chain)
- EBS PVC for MariaDB data persistence
- Secret-based credential injection
- NodePort access with EC2 Security Group configuration
- **Troubleshooting**: CrashLoopBackOff, ImagePullBackOff, CORS, PVC Pending, NodePort blocked

---

## ⚡ Key Commands Used

```bash
# Cluster
eksctl create cluster --name demo-ekscluster --region ap-south-1
aws eks update-kubeconfig --name demo-ekscluster

# Apply manifests
kubectl apply -f <file>.yaml
kubectl delete -f <file>.yaml

# Debugging
kubectl describe pod <pod-name>
kubectl logs <pod-name>
kubectl exec -it <pod-name> -- bash

# Rollout
kubectl rollout status deploy <name>
kubectl rollout history deploy <name>
kubectl rollout undo deploy <name>
kubectl set image deploy/<name> <container>=<image>

# StatefulSet
kubectl get statefulset
kubectl get pvc

# HPA
kubectl get hpa -w
kubectl top pods
kubectl top nodes
kubectl describe hpa <name>

# Ingress
kubectl get ingress
kubectl describe ingress <name>
kubectl logs -n ingress-nginx deployment/ingress-nginx-controller

# Node info
kubectl get nodes -o wide
```

---

> **Author**: Saroj Behera
> **GitHub**: [sarojbehera0610](https://github.com/sarojbehera0610)
> **Portfolio**: [sarojops.cloud](https://sarojops.cloud)