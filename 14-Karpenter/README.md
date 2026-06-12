# 14 — Karpenter: Intelligent Node Provisioner for EKS

## What is Karpenter?
Karpenter is an open-source node provisioner for Kubernetes, built by AWS. It watches for Pending pods and automatically provisions the right EC2 nodes to fit them — faster and smarter than the traditional Cluster Autoscaler.

## Karpenter vs Cluster Autoscaler

| Feature | Cluster Autoscaler | Karpenter |
|---|---|---|
| Node provisioning speed | 10-15 minutes | 60-90 seconds |
| Instance type selection | Fixed ASG type | Picks best fit automatically |
| Spot instance fallback | Manual config | Automatic |
| Bin packing efficiency | Basic | Advanced |
| Node consolidation | ❌ | ✅ merges underused nodes |
| Scale to zero nodes | Limited | ✅ |

## How Karpenter Works
Pod goes Pending (no capacity)
│
▼
Karpenter controller detects Pending pod
│
▼
Reads NodePool CRD (constraints, instance types, zones)
│
▼
Calls EC2 API directly — picks cheapest/fastest fit
│
▼
New node joins cluster in ~60-90 seconds
│
▼
Pending pods scheduled on new node ✅

## Two Core CRDs

### EC2NodeClass
Defines AWS-specific node configuration:
- AMI family (AL2023)
- IAM role for nodes
- Subnet discovery via tags
- Security group discovery via tags

### NodePool
Defines node provisioning constraints:
- Allowed instance types
- Capacity type (on-demand/spot)
- CPU/Memory limits
- Consolidation policy

## Cluster Info
- **EKS Cluster:** ekscluster
- **Region:** ap-south-1
- **K8s Version:** 1.35
- **Karpenter Version:** 1.3.3 (upgraded from 1.0.0)

## Setup Steps Performed

### 1. IAM Role for Karpenter Controller (IRSA)
Created `KarpenterControllerRole-ekscluster` with trust policy pointing to OIDC provider. This allows Karpenter pod to assume the role via web identity token.

### 2. IAM Policy
Created `KarpenterControllerPolicy-ekscluster` with permissions for:
- EC2 instance management (RunInstances, CreateFleet, TerminateInstances)
- SSM parameter reads (for AMI discovery)
- SQS access (for spot interruption handling)
- IAM instance profile management

### 3. Subnet and Security Group Tagging
Tagged all 6 cluster subnets and cluster security group with:
karpenter.sh/discovery = ekscluster
Karpenter uses these tags to auto-discover networking resources.

### 4. aws-auth ConfigMap Update
Added `KarpenterControllerRole-ekscluster` to aws-auth so Karpenter-provisioned nodes can join the cluster:
```yaml
- rolearn: arn:aws:iam::051997448832:role/KarpenterControllerRole-ekscluster
  username: system:node:{{EC2PrivateDNSName}}
  groups:
    - system:bootstrappers
    - system:nodes
```

### 5. SQS Queue
Created SQS queue named `ekscluster` for spot instance interruption handling.

### 6. Helm Install
```bash
helm upgrade --install karpenter oci://public.ecr.aws/karpenter/karpenter \
  --version 1.3.3 \
  --namespace kube-system \
  --set settings.clusterName=ekscluster \
  --set settings.interruptionQueue=ekscluster \
  --set replicas=1 \
  --set serviceAccount.annotations."eks\.amazonaws\.com/role-arn"="arn:aws:iam::051997448832:role/KarpenterControllerRole-ekscluster"
```

## What Worked ✅

| Component | Status |
|---|---|
| OIDC provider registration | ✅ Created via eksctl |
| IAM role + policy | ✅ Created and attached |
| Subnet tagging (6 subnets) | ✅ All tagged |
| Security group tagging | ✅ Tagged |
| aws-auth patched | ✅ Both roles present |
| SQS queue created | ✅ ekscluster queue |
| Karpenter pod running | ✅ 1/1 Running |
| EC2NodeClass | ✅ READY: True |
| NodePool | ✅ READY: True |
| AMI discovery (AL2023/K8s 1.35) | ✅ ami-03b9b13f7a9961c16 |
| Pending pod detection | ✅ Karpenter detects within seconds |

## Issues Encountered and Resolved

### Issue 1 — OIDC Provider Not Registered
**Error:** `No OpenIDConnect provider found in your account`
**Cause:** OIDC provider existed in EKS but was not registered in IAM
**Fix:**
```bash
eksctl utils associate-iam-oidc-provider --cluster ekscluster --region ap-south-1 --approve
```

### Issue 2 — SQS Queue Missing
**Error:** `AWS.SimpleQueueService.NonExistentQueue: The specified queue does not exist`
**Cause:** Karpenter requires an SQS queue for spot interruption handling, not created automatically
**Fix:**
```bash
aws sqs create-queue --queue-name ekscluster --region ap-south-1
```

### Issue 3 — Karpenter 1.0.0 Not Compatible with K8s 1.35
**Error:** `karpenter version is not compatible with K8s version 1.35`
**Cause:** Karpenter 1.0.0 only supports up to K8s 1.32. AL2 AMIs also don't exist for 1.35
**Fix:** Upgraded Karpenter to 1.3.3 and switched AMI family from AL2 to AL2023

### Issue 4 — IAM Policy Missing ec2:CreateFleet on *
**Error:** `Controller isn't authorized to call ec2:CreateFleet`
**Cause:** Initial policy had resource-scoped CreateFleet but needs `Resource: *`
**Fix:** Created new policy version with `ec2:CreateFleet` on `Resource: *`

### Issue 5 — NodeRestriction Plugin Blocking karpenter.k8s.aws Label Domain
**Error:** `label domain "karpenter.k8s.aws" is restricted`
**Cause:** Kubernetes 1.35 NodeRestriction admission plugin introduced stricter label domain restrictions. When Karpenter tries to create a NodeClaim, it automatically injects `karpenter.k8s.aws/ec2nodeclass` as a requirement label — which K8s 1.35 blocks as a restricted domain.
**Status:** Known compatibility issue between Karpenter 1.3.x and EKS 1.35. NodeRestriction is a cluster-level admission plugin and cannot be bypassed via RBAC or configuration.
**Impact:** Node provisioning blocked. Karpenter detects Pending pods correctly and computes NodeClaims but cannot create them.
**Resolution path:** This will be resolved in a future Karpenter release that handles the K8s 1.35 NodeRestriction changes, or by running on EKS 1.32.

## Key Learnings

1. **OIDC must be registered in IAM** — EKS creates the OIDC endpoint but doesn't register it in IAM automatically. `eksctl utils associate-iam-oidc-provider` is required for IRSA to work.

2. **Karpenter version must match K8s version** — Always check the compatibility matrix before installing. Karpenter 1.0.0 → max K8s 1.32. Karpenter 1.3.3 → supports K8s 1.35.

3. **AL2 vs AL2023** — AL2 AMIs stopped at K8s 1.32. For K8s 1.33+, AL2023 is required.

4. **SQS queue is mandatory** — Even for on-demand instances, Karpenter panics at startup without the interruption queue.

5. **aws-auth patch vs edit** — `kubectl patch` is safer than `kubectl edit` for ConfigMap updates, as it avoids YAML indentation errors in vi.

6. **NodeRestriction in K8s 1.35** — Newer Kubernetes versions have tighter security around node label domains. This is a security improvement but creates compatibility friction with controllers that inject custom label domains into NodeClaims.

## Files in this folder

| File | Description |
|---|---|
| `karpenter-nodeclass.yaml` | EC2NodeClass — AL2023 AMI, subnet/SG discovery by tags |
| `karpenter-nodepool.yaml` | NodePool — t3.medium/large/c5.large, on-demand, ap-south-1a/1b |
| `karpenter-test-deploy.yaml` | Test deployment — 5 replicas x 500m CPU to trigger node provisioning |