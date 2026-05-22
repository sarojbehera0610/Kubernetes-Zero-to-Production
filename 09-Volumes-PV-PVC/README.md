# Kubernetes Volumes - PV, PVC, StorageClass, StatefulSet with Dynamic and Static Provisioning

## What is a Volume in Kubernetes

A Volume in Kubernetes is a storage unit attached to a Pod. Unlike container storage, volumes persist data even if the container restarts. Kubernetes supports many volume types like hostPath, emptyDir, awsElasticBlockStore, and persistent volumes.

---

## Key Concepts

### PersistentVolume (PV)
A PV is a piece of storage provisioned in the cluster either manually (static) or automatically (dynamic). It exists independently of any Pod. It has its own lifecycle.

### PersistentVolumeClaim (PVC)
A PVC is a request for storage by a user. It specifies size, accessMode, and storageClassName. Kubernetes binds a PVC to a matching PV automatically.

### StorageClass (SC)
A StorageClass defines how storage is dynamically provisioned. It references a provisioner like `ebs.csi.aws.com` which automatically creates the actual disk when a PVC is created.

### ReclaimPolicy
- **Retain** - PV data is kept even after PVC is deleted. Manual cleanup required.
- **Delete** - PV and the actual disk are deleted automatically when PVC is deleted.

### volumeBindingMode
- **WaitForFirstConsumer** - EBS volume is not created until a Pod actually requests it. Ensures the volume is created in the same AZ as the Pod.
- **Immediate** - Volume is created as soon as PVC is created.

### AccessModes
- **ReadWriteOnce (RWO)** - Volume can be mounted by one node at a time.
- **ReadWriteMany (RWX)** - Volume can be mounted by multiple nodes simultaneously.

---

## Dynamic Provisioning

In dynamic provisioning the actual disk (EBS volume) is created automatically by the StorageClass provisioner when a PVC is bound.

### Connection Chain
StorageClass (provisioner: ebs.csi.aws.com)
↑ storageClassName referenced by
PVC (requests storage)
↑ PVC name or volumeClaimTemplates referenced by
Pod/StatefulSet container
↑ volumeMounts.name matches volumes.name or volumeClaimTemplates.name

### Prerequisites
- EKS cluster with EBS CSI Driver addon installed
- Node group IAM role must have AmazonEBSCSIDriverPolicy attached

### Install EBS CSI Driver

```bash
# Get node group name
aws eks list-nodegroups --cluster-name demo-ekscluster --region ap-south-1

# Get node IAM role
aws eks describe-nodegroup \
  --cluster-name demo-ekscluster \
  --nodegroup-name <nodegroup-name> \
  --region ap-south-1 \
  --query "nodegroup.nodeRole" \
  --output text

# Attach EBS CSI policy to node role
aws iam attach-role-policy \
  --role-name <role-name> \
  --policy-arn arn:aws:iam::aws:policy/service-role/AmazonEBSCSIDriverPolicy

# Install addon
eksctl create addon \
  --name aws-ebs-csi-driver \
  --cluster demo-ekscluster \
  --region ap-south-1 \
  --force

# Restart driver
kubectl rollout restart deployment ebs-csi-controller -n kube-system

# Verify
kubectl get pods -n kube-system | grep ebs
```

### sc.yaml
```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: storage-ebs
provisioner: ebs.csi.aws.com
parameters:
  type: gp3
reclaimPolicy: Retain
volumeBindingMode: WaitForFirstConsumer
```

### pvc.yaml (standalone - for single pod use)
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-pvc
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: storage-ebs
  resources:
    requests:
      storage: 2Gi
```

### sts-with-pvc.yaml (StatefulSet referencing standalone PVC)
```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-sts-service-1
  labels:
    app: my-sts
spec:
  ports:
    - port: 3306
      name: web
  clusterIP: None
  selector:
    app: my-sts
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web-sts-1
spec:
  selector:
    matchLabels:
      app: my-sts
  serviceName: "my-sts-service-1"
  replicas: 1
  template:
    metadata:
      labels:
        app: my-sts
    spec:
      containers:
        - name: container-1
          image: mysql:8
          env:
            - name: MYSQL_ROOT_PASSWORD
              value: saroj123
          volumeMounts:
            - name: sql-storage-1
              mountPath: /var/lib/mysql
      volumes:
        - name: sql-storage-1
          persistentVolumeClaim:
            claimName: my-pvc
```

### sts-without-pvc.yaml (StatefulSet with volumeClaimTemplates - auto creates PVC per replica)
```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-sts-service-1
  labels:
    app: my-sts
spec:
  ports:
    - port: 3306
      name: web
  clusterIP: None
  selector:
    app: my-sts
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web-sts-1
spec:
  selector:
    matchLabels:
      app: my-sts
  serviceName: "my-sts-service-1"
  replicas: 3
  template:
    metadata:
      labels:
        app: my-sts
    spec:
      containers:
        - name: container-1
          image: mysql:8
          env:
            - name: MYSQL_ROOT_PASSWORD
              value: saroj123
          volumeMounts:
            - name: sql-storage-1
              mountPath: /var/lib/mysql
  volumeClaimTemplates:
    - metadata:
        name: sql-storage-1
      spec:
        accessModes: ["ReadWriteOnce"]
        storageClassName: storage-ebs
        resources:
          requests:
            storage: 2Gi
```

### Apply Dynamic Setup
```bash
kubectl apply -f sc.yaml
kubectl get sc

# Approach 1 - with standalone PVC
kubectl apply -f pvc.yaml
kubectl get pvc
kubectl apply -f sts-with-pvc.yaml

# Approach 2 - without standalone PVC
kubectl apply -f sts-without-pvc.yaml

kubectl get pods -w
kubectl get pvc
kubectl get pv
kubectl describe pvc <pvc-name>
kubectl exec -it web-sts-1-0 -- ls /var/lib/mysql
```

---

## What Happens with replicas:3 and Shared PVC

When sts-with-pvc is used with replicas:3 all pods share one PVC and one EBS volume. Kubernetes schedules all pods on the same node because ReadWriteOnce allows one node at a time. Multiple MySQL processes writing to the same data directory causes data corruption and repeated pod restarts.

```bash
kubectl get pods -o wide
# All pods will be on same node

kubectl logs <pod-name> --previous
# InnoDB crash or lock errors visible
```

This proves why volumeClaimTemplates exists. Each replica must have its own dedicated PVC and EBS volume.

---

## Retain Policy - Reusing Released PVs

When PVC is deleted with reclaimPolicy Retain, the PV moves to Released status. Data is safe but Kubernetes will not rebind it automatically. To reuse:

```bash
kubectl edit pv <pv-name>
# Remove the entire claimRef block from spec
# Save and exit

kubectl get pv
# Status changes from Released to Available

# Now apply new PVC - it will bind to this PV
kubectl apply -f pvc.yaml
```

---

## Static Provisioning

In static provisioning you create PVs manually. No StorageClass provisioner needed. StorageClass name in PV and PVC just acts as a matching label.

### Connection Chain
PV (created manually with storageClassName: static-sc)
↑ matched by storageClassName + size + accessMode
PVC auto-created by StatefulSet volumeClaimTemplates
↑ volumeMounts.name matches volumeClaimTemplates.name
Container

### pv.yaml
```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: static-pv-1
spec:
  capacity:
    storage: 2Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: static-sc
  hostPath:
    path: /mnt/data/pv1
    type: DirectoryOrCreate
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: static-pv-2
spec:
  capacity:
    storage: 2Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: static-sc
  hostPath:
    path: /mnt/data/pv2
    type: DirectoryOrCreate
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: static-pv-3
spec:
  capacity:
    storage: 2Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: static-sc
  hostPath:
    path: /mnt/data/pv3
    type: DirectoryOrCreate
```

### sts-static.yaml
```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-sts-service-static
  labels:
    app: my-sts-static
spec:
  ports:
    - port: 3306
      name: web
  clusterIP: None
  selector:
    app: my-sts-static
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web-sts-static
spec:
  selector:
    matchLabels:
      app: my-sts-static
  serviceName: "my-sts-service-static"
  replicas: 3
  template:
    metadata:
      labels:
        app: my-sts-static
    spec:
      containers:
        - name: container-1
          image: mysql:8
          env:
            - name: MYSQL_ROOT_PASSWORD
              value: saroj123
          volumeMounts:
            - name: sql-storage-static
              mountPath: /var/lib/mysql
  volumeClaimTemplates:
    - metadata:
        name: sql-storage-static
      spec:
        accessModes: ["ReadWriteOnce"]
        storageClassName: static-sc
        resources:
          requests:
            storage: 2Gi
```

### Apply Static Setup
```bash
kubectl apply -f pv.yaml
kubectl get pv
# All 3 should show Available

kubectl apply -f sts-static.yaml
kubectl get pods -w
kubectl get pvc
kubectl get pv
# PVs should show Bound

kubectl exec -it web-sts-static-0 -- ls /var/lib/mysql
# MySQL data files confirm volume is mounted and working
```

---

## Dynamic vs Static - Key Difference

Dynamic - StorageClass provisioner creates the actual disk automatically when PVC is created. No manual disk management needed. Used in production with cloud storage like EBS.

Static - You create PVs manually pointing to existing storage. No provisioner involved. StorageClass name is just a label for matching. Used when you have pre-existing storage or for local/hostPath volumes.