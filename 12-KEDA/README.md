# 12 — KEDA (Kubernetes Event-Driven Autoscaling)

## What is KEDA?
KEDA extends Kubernetes HPA to support event-driven autoscaling. While HPA only scales on CPU/Memory, KEDA can scale pods based on 60+ external sources — queues, Kafka, Prometheus, Cron schedules, and more. Most importantly, KEDA supports **scale-to-zero**, which HPA cannot do.

## Architecture
KEDA installs 3 components into the cluster:
- **keda-operator** — watches ScaledObject CRDs and manages scaling decisions
- **keda-metrics-apiserver** — exposes external metrics to the K8s metrics API
- **keda-admission-webhooks** — validates ScaledObject configs before applying

## KEDA vs HPA

| Feature | HPA | KEDA |
|---|---|---|
| Scale-to-zero | ❌ | ✅ |
| CPU/Memory triggers | ✅ | ✅ |
| Queue/Event triggers | ❌ | ✅ |
| Cron-based scaling | ❌ | ✅ |
| Auto-manages HPA | — | ✅ |

## Key Resource: ScaledObject
ScaledObject is the KEDA-specific CRD that links a Deployment to a trigger source. KEDA auto-creates and manages the underlying HPA — you never touch it directly.

## Installation
```bash
helm repo add kedacore https://kedacore.github.io/charts
helm repo update
kubectl create namespace keda
helm install keda kedacore/keda --namespace keda
kubectl get pods -n keda
```

## Files in this folder

| File | Description |
|---|---|
| `keda-demo-deploy.yaml` | nginx:alpine Deployment used as the demo workload |
| `keda-scaledobject-cpu.yaml` | ScaledObject with CPU trigger — scales 1→5 pods when CPU > 50% |
| `keda-scaledobject-cron.yaml` | ScaledObject with Cron trigger — scales to 0 at night, 3 during work hours |

## Demo 1 — CPU Based Scaling

### What was tested
- Deployed nginx workload with CPU requests/limits set
- Created ScaledObject with CPU trigger at 50% threshold
- KEDA auto-created HPA `keda-hpa-keda-demo-scaledobject`
- Generated load using multiple busybox pods hammering nginx

### Result
cpu: 29%/50%  → Replicas: 1
cpu: 82%/50%  → Replicas: 1  (threshold crossed)
cpu: 105%/50% → Replicas: 2  (scaled up)
cpu: 119%/50% → Replicas: 2
cpu: 81%/50%  → Replicas: 3
cpu: 55%/50%  → Replicas: 5  (hit max)
cpu: 0%/50%   → Replicas: 1  (scaled back down after load stopped)

### Key learnings
- CPU trigger requires `minReplicaCount: 1` — cannot scale to zero because there must be a pod running to measure CPU
- KEDA admission webhook blocked `minReplicaCount: 0` with CPU trigger and gave a clear error
- `pollingInterval` and `cooldownPeriod` only apply when `minReplicaCount: 0`

## Demo 2 — Cron Based Scaling (Scale to Zero)

### What was tested
- Created ScaledObject with Cron trigger (Asia/Kolkata timezone)
- Set a specific time window to scale UP to 3 pods and DOWN to 0 pods
- Observed full cycle: 0 → 3 → 0

### Result
Outside cron window  → Pods: 0  (scale to zero!)
Cron start time hit  → Pods: 3  (scaled up automatically)
Cron end time hit    → Pods: 0  (scaled back to zero)

### Key learnings
- Cron trigger supports `minReplicaCount: 0` — true scale-to-zero
- KEDA cron reconciliation has ~2-5 minute delay after end time — normal behavior
- Real-world use: scale to 0 at night (8 PM) and back up at 9 AM on weekdays — saves EC2 costs significantly

## Cluster Info
- **EKS Cluster:** demo-ekscluster
- **Region:** ap-south-1
- **KEDA Version:** 2.20.1
- **Helm Chart:** kedacore/keda