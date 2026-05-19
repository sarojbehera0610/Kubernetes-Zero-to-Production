# Kubernetes Zero to Production 🚀

A hands-on learning repository documenting my Kubernetes journey from core concepts
to production-grade workloads on AWS EKS. Each folder contains topic-specific
manifest files, notes, and real outputs from a self-managed EKS cluster on AWS EC2
(ap-south-1).

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
```

---

## 📌 Upcoming Topics

- ConfigMaps & Secrets
- Persistent Volumes & PVC
- Ingress
- RBAC
- Helm
- Terraform for EKS
- GitHub Actions CI/CD
- Prometheus & Grafana

---

> **Author**: Saroj Behera
> **GitHub**: [sarojbehera0610](https://github.com/sarojbehera0610)
> **Portfolio**: [sarojops.cloud](https://sarojops.cloud)