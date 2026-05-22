# Kubernetes DaemonSet

## What is a DaemonSet

A DaemonSet ensures that a copy of a Pod runs on every node in the cluster. When a new node is added to the cluster a Pod is automatically scheduled on it. When a node is removed the Pod is garbage collected. DaemonSets are used for node-level operations that need to run everywhere.

---

## Use Cases

- Log collection agents (Fluentd, Filebeat)
- Monitoring agents (Prometheus Node Exporter, Datadog)
- Network plugins (Calico, Weave)
- Storage drivers (EBS CSI node pods run as DaemonSet)
- Security agents

---

## DaemonSet vs Deployment

Deployment - You specify replica count. Pods run on any available nodes based on scheduler decision.

DaemonSet - No replica count. One Pod per node always. Scheduler does not decide placement.

---

## daemonset.yaml
```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: my-daemonset
  labels:
    app: my-ds
spec:
  selector:
    matchLabels:
      app: my-ds
  template:
    metadata:
      labels:
        app: my-ds
    spec:
      containers:
        - name: container-1
          image: nginx:latest
```

---

## Apply and Verify
```bash
kubectl apply -f daemonset.yaml

kubectl get daemonset
# Shows DESIRED = number of nodes, CURRENT = running pods

kubectl get pods -o wide
# One pod per node confirmed

kubectl describe daemonset my-daemonset
```

---

## Key Observations

Number of pods created equals number of worker nodes in the cluster. You cannot scale a DaemonSet manually. If you add a new node a pod automatically appears on it. The ebs-csi-node pods you saw during EBS CSI driver setup were also running as a DaemonSet - one on each node.