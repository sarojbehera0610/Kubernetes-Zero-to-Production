# 13 — Node Affinity

## What is Node Affinity?
Node Affinity allows you to control which nodes your pods are scheduled on, based on node labels. It is the advanced version of `nodeSelector` with support for OR/NOT conditions, soft preferences, and multiple expressions.

## Node Affinity vs nodeSelector

| Feature | nodeSelector | Node Affinity |
|---|---|---|
| Basic label matching | ✅ | ✅ |
| OR conditions | ❌ | ✅ |
| NOT conditions | ❌ | ✅ |
| Soft preference | ❌ | ✅ |
| Multiple expressions | ❌ | ✅ |

## Two Types

### 1. requiredDuringSchedulingIgnoredDuringExecution
Hard rule — pod will NOT schedule unless a matching node exists. Pod stays Pending forever if no match found.

### 2. preferredDuringSchedulingIgnoredDuringExecution
Soft rule — Kubernetes tries to schedule on matching node but schedules anywhere if no match available. Uses a `weight` (1-100) to indicate preference strength.

## IgnoredDuringExecution
Both types have this suffix — means once a pod is running, if the node's labels change, the pod is NOT evicted. Affinity rules only apply at scheduling time.

## Operators

| Operator | Meaning |
|---|---|
| `In` | label value must be in the list |
| `NotIn` | label value must NOT be in the list |
| `Exists` | label key must exist (any value) |
| `DoesNotExist` | label key must not exist |
| `Gt` | label value must be greater than |
| `Lt` | label value must be less than |

## Files in this folder

| File | Description |
|---|---|
| `node-affinity-required.yaml` | Hard rule — pod must land on node-type=worker |
| `node-affinity-pending.yaml` | Hard rule — pod stays Pending, no node-type=gpu exists |
| `node-affinity-preferred.yaml` | Soft rule — pod prefers worker node, weight=80 |
| `node-affinity-notin.yaml` | NotIn operator — pod avoids node-type=infra |
| `node-affinity-combined.yaml` | Combined hard + soft rules (production pattern) |

## Node Setup Used
```bash
# Label nodes before running demos
kubectl label node  node-type=worker
kubectl label node  node-type=infra

# Remove labels after demo
kubectl label node  node-type-
kubectl label node  node-type-
```

## Demo Results

### Demo 1 — Required Affinity (Hard Rule)
Pod: pod-required-affinity
Rule: node-type In [worker]
Result: Scheduled on worker node ✅

### Demo 2 — Required Affinity with No Match
Pod: pod-no-match
Rule: node-type In [gpu]
Result: Pending forever — no node has node-type=gpu ⚠️
Event: 0/2 nodes are available: 2 node(s) didn't match Pod's node affinity/selector

### Demo 3 — Preferred Affinity (Soft Rule)
Pod: pod-preferred-affinity
Rule: prefer node-type=worker, weight=80
Result: Scheduled on worker node ✅
Note: Would schedule anywhere if worker node unavailable

### Demo 4 — NotIn Anti-Affinity
Pod: pod-avoid-infra
Rule: node-type NotIn [infra]
Result: Scheduled on worker node, avoided infra node ✅

### Demo 5 — Combined Required + Preferred
Pod: pod-combined-affinity
Hard rule: kubernetes.io/os=linux (all nodes qualify)
Soft rule: prefer node-type=worker, weight=80
Result: Scheduled on worker node ✅
Pattern: Hard rule filters pool, soft rule picks best within pool

## Key Learnings
- Wrong label in required rule = pod stuck Pending forever with no obvious error
- Preferred rule never blocks scheduling — always falls back
- NotIn is used for anti-affinity (avoid specific nodes)
- Combined pattern is most common in production
- Node labels are custom — you define them based on your node types

## Cluster Info
- EKS Cluster: demo-ekscluster
- Region: ap-south-1
- Nodes: 2x c7i-flex.large (ap-south-1a, ap-south-1b)