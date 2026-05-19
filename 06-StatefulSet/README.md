# StatefulSet

Used for stateful applications that require stable identity and
persistent storage per pod. Examples: MySQL, MongoDB, Kafka, ZooKeeper.

## StatefulSet vs Deployment

| Feature | Deployment | StatefulSet |
|---|---|---|
| Pod names | Random suffix | Ordered: pod-0, pod-1, pod-2 |
| Pod identity | Interchangeable | Unique and stable |
| Storage | Shared or none | Each pod gets own PVC |
| Start order | Parallel | Sequential (0 → 1 → 2) |
| Delete order | Parallel | Reverse (2 → 1 → 0) |
| Use case | Stateless apps | Databases, distributed systems |

## StatefulSet Manifest (without PVC)
```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: sts-headless
spec:
  serviceName: headless-svc
  replicas: 3
  selector:
    matchLabels:
      app: sts-headless
  template:
    metadata:
      labels:
        app: sts-headless
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
  name: headless-svc
spec:
  clusterIP: None
  selector:
    app: sts-headless
  ports:
    - port: 80
      targetPort: 80
```

## Headless vs Normal Service — DNS Test

### What we tested
Ran `nslookup` inside pods of two StatefulSets:
- `sts-headless` using `clusterIP: None`
- `sts-normal` using a regular Service

### Headless result
nslookup headless-svc
Address: 192.168.33.65   ← pod-0 actual IP
Address: 192.168.41.128  ← pod-1 actual IP
Address: 192.168.61.141  ← pod-2 actual IP
Returns all individual Pod IPs directly.

### Normal result
nslookup normal-svc
Address: 10.100.237.227  ← single virtual ClusterIP
Returns only one virtual IP — you cannot target individual pods.

### Individual Pod DNS
```bash
# headless — works ✅
nslookup sts-headless-0.headless-svc.default.svc.cluster.local
# returns pod-0 IP directly

# normal — returns IP but no stable per-pod identity ❌
nslookup sts-normal-0.normal-svc.default.svc.cluster.local
```

### Why headless is required
In databases like MySQL:
- pod-0 = Primary (writes only)
- pod-1, pod-2 = Replicas (reads only)

With normal service, traffic is load balanced randomly — a write
could go to a replica and corrupt data. Headless lets the app
target pod-0 directly by DNS name.

## PVC Note
StatefulSet `volumeClaimTemplates` automatically creates a PVC per pod.
Requires a StorageClass or manually created PVs. For hostPath testing:
```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: my-pv-0
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: /tmp/my-storage-0
```

## Key Learning
- `serviceName` is a required field in StatefulSet spec
- Headless = `clusterIP: None` — DNS returns Pod IPs, not virtual IP
- Without PV/StorageClass, volumeClaimTemplates will keep pod in Pending