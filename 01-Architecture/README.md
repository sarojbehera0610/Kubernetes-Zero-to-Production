# Kubernetes Architecture

## Control Plane Components

| Component | Role |
|---|---|
| API Server | Entry point for all kubectl commands and cluster communication |
| etcd | Distributed key-value store — stores entire cluster state |
| Scheduler | Assigns pods to worker nodes based on resource availability |
| Controller Manager | Watches cluster state and reconciles to desired state |
| cloud-controller-manager | Integrates with AWS/GCP/Azure (ELB, EBS, node lifecycle) |

## Worker Node Components

| Component | Role |
|---|---|
| kubelet | Agent on each node — communicates with API server, manages pods |
| kube-proxy | Handles networking rules, service routing on each node |
| Container Runtime | Runs containers (containerd / Docker) |

## Flow: What happens when you run `kubectl apply`
kubectl → API Server → etcd (stores desired state)
→ Scheduler (assigns pod to node)
→ kubelet on node (pulls image, starts container)

## Key Learning
- Control plane = brain of the cluster
- Worker node = muscle that runs actual workloads
- etcd = single source of truth — if etcd is lost, cluster state is lost