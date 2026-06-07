# 10 — Horizontal Pod Autoscaler (HPA) 📈

Hands-on HPA implementation on AWS EKS — automatically scaling pods based on CPU
utilisation using the Metrics Server. Tested with a live load generator to trigger
real scale-out events.

---

## 📖 Concept

### What is HPA?

HPA (Horizontal Pod Autoscaler) automatically increases or decreases the number of
pod **replicas** in a Deployment (or ReplicaSet / StatefulSet) based on observed
resource metrics — primarily CPU and memory utilisation.

```
              ┌──────────────┐
              │  Metrics     │  ← collects pod CPU/memory
              │  Server      │
              └──────┬───────┘
                     │ reports metrics
              ┌──────▼───────┐
              │     HPA      │  ← compares vs target utilisation
              │  Controller  │
              └──────┬───────┘
                     │ scales
              ┌──────▼───────┐
              │  Deployment  │  ← replica count adjusted
              └──────────────┘
```

### HPA Scaling Formula

```
desiredReplicas = ceil(currentReplicas × (currentMetricValue / desiredMetricValue))
```

Example: 1 pod at 90% CPU, target 50% → ceil(1 × 90/50) = ceil(1.8) = **2 pods**

### Key Fields in HPA Spec

| Field | Description |
|---|---|
| `scaleTargetRef` | Points to the Deployment to scale |
| `minReplicas` | Minimum pods (default: 1) |
| `maxReplicas` | Upper limit on pod count |
| `averageUtilization` | Target CPU % of the pod's `requests.cpu` |

> ⚠️ **Critical**: HPA only works if `resources.requests.cpu` is defined in the
> container spec. Without it, HPA cannot calculate utilisation and stays stuck.

---

## 📋 Prerequisites

### 1. Install Metrics Server on EKS

Metrics Server is **not installed by default** on EKS. Without it, HPA shows
`<unknown>` for current metrics and never scales.

```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

Verify it is running:

```bash
kubectl get deployment metrics-server -n kube-system
# NAME             READY   UP-TO-DATE   AVAILABLE
# metrics-server   1/1     1            1

kubectl top nodes
# NAME                          CPU(cores)   CPU%   MEMORY(bytes)   MEMORY%
# ip-192-168-xx-xx.ap-south-1   22m          1%     512Mi           26%
```

---

## 📁 Manifest Files

| File | Purpose |
|---|---|
| `hpa-deployment.yaml` | Deployment with CPU requests/limits defined |
| `hpa-service.yaml` | ClusterIP Service to expose the deployment internally |
| `hpa.yaml` | HPA resource — min 1, max 5 replicas, target 50% CPU |
| `load-generator.yaml` | BusyBox pod that generates continuous HTTP load |

---

## 🚀 Step-by-Step Practical

### Step 1 — Apply the Deployment

```bash
kubectl apply -f hpa-deployment.yaml
kubectl get pods
# NAME                                  READY   STATUS    RESTARTS
# hpa-demo-deployment-xxxxxxxxx-xxxxx   1/1     Running   0
```

### Step 2 — Apply the Service

```bash
kubectl apply -f hpa-service.yaml
kubectl get svc hpa-demo-service
# NAME               TYPE        CLUSTER-IP      PORT(S)
# hpa-demo-service   ClusterIP   10.100.xx.xx    80/TCP
```

### Step 3 — Apply the HPA

```bash
kubectl apply -f hpa.yaml
kubectl get hpa
# NAME           REFERENCE                        TARGETS   MINPODS   MAXPODS   REPLICAS
# hpa-demo-hpa   Deployment/hpa-demo-deployment   0%/50%    1         5         1
```

> If you see `<unknown>/50%` instead of `0%/50%` — Metrics Server is not ready yet.
> Wait 60–90 seconds and check again.

### Step 4 — Monitor CPU in real time (separate terminal)

```bash
# Watch HPA status live
kubectl get hpa -w

# Watch pods scaling up/down
kubectl get pods -w
```

### Step 5 — Start the Load Generator

```bash
kubectl apply -f load-generator.yaml
```

This pod hits `http://hpa-demo-service` in a tight loop — CPU spikes quickly.

### Step 6 — Observe Scale-Out

Within ~30–60 seconds, HPA detects CPU crossing 50% and starts scaling:

```bash
kubectl get hpa
# NAME           REFERENCE                        TARGETS    MINPODS   MAXPODS   REPLICAS
# hpa-demo-hpa   Deployment/hpa-demo-deployment   128%/50%   1         5         3

kubectl get pods
# NAME                                  READY   STATUS    RESTARTS
# hpa-demo-deployment-xxxxxxxxx-aaa     1/1     Running   0
# hpa-demo-deployment-xxxxxxxxx-bbb     1/1     Running   0
# hpa-demo-deployment-xxxxxxxxx-ccc     1/1     Running   0
# load-generator                        1/1     Running   0
```

### Step 7 — Stop Load and Observe Scale-In

```bash
kubectl delete pod load-generator
```

HPA has a **cool-down period of ~5 minutes** before scaling back down to avoid
thrashing. After the cool-down:

```bash
kubectl get hpa
# NAME           REFERENCE                        TARGETS   MINPODS   MAXPODS   REPLICAS
# hpa-demo-hpa   Deployment/hpa-demo-deployment   0%/50%    1         5         1
```

Pods scale back to 1 automatically.

---

## 🐛 Errors Faced & Troubleshooting

### ❌ Error 1 — `<unknown>/50%` in HPA TARGETS

**Symptom:**
```bash
kubectl get hpa
# NAME           TARGETS         REPLICAS
# hpa-demo-hpa   <unknown>/50%   1
```

**Cause:** Metrics Server not installed or not yet ready.

**Fix:**
```bash
# Install Metrics Server
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

# Wait for it to be ready
kubectl rollout status deployment/metrics-server -n kube-system

# On EKS, if it stays in Pending — patch with hostNetwork
kubectl patch deployment metrics-server -n kube-system \
  --type=json \
  -p='[{"op":"add","path":"/spec/template/spec/hostNetwork","value":true}]'
```

---

### ❌ Error 2 — HPA stuck at 1 replica despite high load

**Symptom:** CPU was clearly high but `REPLICAS` stayed at 1.

**Cause:** `resources.requests.cpu` was missing from the container spec. HPA uses
`requests.cpu` as the baseline for utilisation calculation — if it is absent,
the controller cannot compute a ratio and makes no scaling decision.

**Fix:** Add resource requests to the container:
```yaml
resources:
  requests:
    cpu: "100m"
    memory: "128Mi"
```

Then re-apply the deployment. HPA begins showing real percentages immediately.

---

### ❌ Error 3 — Pods in `Pending` after scale-out

**Symptom:** HPA increased replicas but new pods stayed `Pending`.

**Cause:** t2.micro nodes ran out of allocatable CPU. Two nodes × ~2 CPUs but
already running system pods consumed most capacity.

**Fix:** Either delete idle pods to free resources, or (on a real cluster) add
more nodes via the ASG. For this lab, we deleted the load generator immediately
after observing the scale-out to let pods settle.

---

## 📌 Key Takeaways

- HPA requires **Metrics Server** — install it first on EKS
- Always define **`resources.requests.cpu`** in pod spec — HPA is blind without it
- Scale-out is fast (~30–60s); scale-in has a **5-minute cool-down** by design
- `kubectl get hpa -w` is your best friend during live testing
- On small clusters (t2.micro), Pending pods during scale-out are normal — node
  capacity is the bottleneck, not HPA logic

---

## ⚡ Commands Reference

```bash
# Apply all HPA resources
kubectl apply -f hpa-deployment.yaml
kubectl apply -f hpa-service.yaml
kubectl apply -f hpa.yaml

# Monitor
kubectl get hpa -w
kubectl get pods -w
kubectl top pods

# Load test
kubectl apply -f load-generator.yaml
kubectl delete pod load-generator

# Describe HPA for events and conditions
kubectl describe hpa hpa-demo-hpa

# Cleanup
kubectl delete -f hpa.yaml
kubectl delete -f hpa-deployment.yaml
kubectl delete -f hpa-service.yaml
```

---

> **Author**: Saroj Behera | [sarojops.cloud](https://sarojops.cloud) | [GitHub](https://github.com/sarojbehera0610)