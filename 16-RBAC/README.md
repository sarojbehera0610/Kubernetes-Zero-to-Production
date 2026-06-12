# 16 — RBAC (Role-Based Access Control)

## What is RBAC?
RBAC controls **who can do what on which resources in which namespace** inside a Kubernetes cluster. Without RBAC, anyone with cluster access can read secrets, delete deployments, or modify any namespace — dangerous in real teams.

## The Core Question RBAC Answers
> "Can this **user/serviceaccount** perform this **action** on this **resource** in this **namespace**?"

## 4 Core RBAC Objects

### Role
Defines allowed actions on resources — **namespace scoped**.
```yaml
kind: Role
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list", "watch"]   # read-only
```

### ClusterRole
Same as Role but **cluster-wide** — for cross-namespace or cluster-level resources (nodes, PVs, namespaces).

### RoleBinding
Connects a Role to a Subject (User/ServiceAccount) — in a specific namespace.

### ClusterRoleBinding
Connects a ClusterRole to a Subject — cluster-wide.

## RBAC Model
Subject          +    Role/ClusterRole    =    Access
(Who)                 (What is allowed)        (Granted)
ServiceAccount   →    RoleBinding         →    Namespace access
ServiceAccount   →    ClusterRoleBinding  →    Cluster-wide access

## Verbs — Available Actions

| Verb | What it does |
|---|---|
| `get` | Read one specific resource |
| `list` | Read all resources of a type |
| `watch` | Stream real-time changes |
| `create` | Create new resource |
| `update` | Full update of resource |
| `patch` | Partial update |
| `delete` | Delete one resource |
| `deletecollection` | Delete all resources of a type |

## apiGroups — Resource Categories

| apiGroup | Resources |
|---|---|
| `""` (core) | pods, services, configmaps, secrets, nodes |
| `"apps"` | deployments, daemonsets, statefulsets, replicasets |
| `"rbac.authorization.k8s.io"` | roles, rolebindings |
| `"batch"` | jobs, cronjobs |
| `"networking.k8s.io"` | ingresses, networkpolicies |

## Files in this folder

| File | Description |
|---|---|
| `rbac-role-dev.yaml` | Role — read-only pods/deployments in dev namespace |
| `rbac-rolebinding-dev.yaml` | RoleBinding — binds dev-reader Role to dev-user ServiceAccount |
| `rbac-clusterrole-monitor.yaml` | ClusterRole — read all resources cluster-wide (monitoring pattern) |
| `rbac-clusterrolebinding-monitor.yaml` | ClusterRoleBinding — binds cluster-monitor to monitor-user |
| `rbac-cicd.yaml` | ServiceAccount + Role + RoleBinding — CI/CD pipeline pattern |

## Demo 1 — Role + RoleBinding (Namespace Scoped)

### Setup
```bash
kubectl create namespace dev
kubectl create serviceaccount dev-user -n dev
kubectl apply -f rbac-role-dev.yaml
kubectl apply -f rbac-rolebinding-dev.yaml
```

### What was created
- Role `dev-reader` in `dev` namespace — allows get/list/watch on pods and deployments
- RoleBinding `dev-user-binding` — connects dev-reader to dev-user ServiceAccount

### Verification
```bash
kubectl auth can-i list pods -n dev --as=system:serviceaccount:dev:dev-user
# yes ✅

kubectl auth can-i delete pods -n dev --as=system:serviceaccount:dev:dev-user
# no ✅ (delete not in Role)

kubectl auth can-i list pods -n prod --as=system:serviceaccount:dev:dev-user
# no ✅ (Role is namespace-scoped to dev only)
```

## Demo 2 — ClusterRole + ClusterRoleBinding (Cluster-Wide)

### Use case
Monitoring tool (Prometheus, Datadog) needs to read all resources across all namespaces.

### Setup
```bash
kubectl create serviceaccount monitor-user -n dev
kubectl apply -f rbac-clusterrole-monitor.yaml
kubectl apply -f rbac-clusterrolebinding-monitor.yaml
```

### Verification
```bash
kubectl auth can-i list nodes --as=system:serviceaccount:dev:monitor-user
# yes ✅ (ClusterRole allows cluster-wide node access)
```

## Demo 3 — CI/CD Pipeline ServiceAccount (Production Pattern)

### Use case
CI/CD pipeline (Jenkins, GitHub Actions, ArgoCD) needs to deploy to `prod` namespace but nothing else.

### What was created (single file — rbac-cicd.yaml)
- ServiceAccount `cicd-pipeline` in `prod` namespace
- Role `cicd-deployer` — allows create/update deployments, services, configmaps in prod
- RoleBinding `cicd-binding` — connects cicd-deployer to cicd-pipeline

### Verification
```bash
kubectl auth can-i create deployments -n prod --as=system:serviceaccount:prod:cicd-pipeline
# yes ✅

kubectl auth can-i delete deployments -n prod --as=system:serviceaccount:prod:cicd-pipeline
# no ✅ (delete not in Role — prevents accidental teardown)

kubectl auth can-i list pods -n dev --as=system:serviceaccount:prod:cicd-pipeline
# no ✅ (completely isolated to prod namespace)
```

## Demo 4 — kubectl auth can-i (Access Verification)

`kubectl auth can-i` is the most important RBAC debugging command:

```bash
# Basic syntax
kubectl auth can-i <verb> <resource> -n <namespace> --as=<subject>

# Check your own permissions
kubectl auth can-i --list -n dev

# Check as a ServiceAccount
kubectl auth can-i list pods -n dev \
  --as=system:serviceaccount:dev:dev-user

# Check as a specific user
kubectl auth can-i create deployments \
  --as=jane@company.com -n production
```

## Full Verification Results

| Subject | Action | Namespace | Result | Reason |
|---|---|---|---|---|
| dev-user | list pods | dev | ✅ yes | Role allows |
| dev-user | delete pods | dev | ❌ no | delete not in Role |
| dev-user | list pods | prod | ❌ no | Role scoped to dev only |
| monitor-user | list nodes | cluster | ✅ yes | ClusterRole allows |
| cicd-pipeline | create deployments | prod | ✅ yes | Role allows |
| cicd-pipeline | delete deployments | prod | ❌ no | delete not in Role |
| cicd-pipeline | list pods | dev | ❌ no | Role scoped to prod only |

## Role vs ClusterRole — When to Use Which

| Scenario | Use |
|---|---|
| Developer access to one namespace | Role + RoleBinding |
| Monitoring tool reading all namespaces | ClusterRole + ClusterRoleBinding |
| CI/CD deploying to specific namespace | Role + RoleBinding |
| Admin access to nodes/PVs | ClusterRole + ClusterRoleBinding |
| Reading secrets in one namespace | Role + RoleBinding |

## Key Learnings

1. **Principle of Least Privilege** — always give minimum permissions needed. Never use `verbs: ["*"]` in production.

2. **ServiceAccounts over Users** — in Kubernetes, applications and automation use ServiceAccounts. Human users are managed via IAM (in EKS) or certificates.

3. **`kubectl auth can-i --list`** — run this to see all permissions a subject has. Essential for debugging access issues.

4. **Role is namespace-scoped** — a Role in `dev` gives zero access to `prod`. Always verify with `auth can-i` after creating bindings.

5. **ClusterRole can be bound with RoleBinding** — you can bind a ClusterRole using a RoleBinding to limit it to one namespace. Useful for reusing common role definitions.

6. **`system:serviceaccount:namespace:name`** — this is the full format for referencing a ServiceAccount as a subject in `--as` flag.

## Cluster Info
- EKS Cluster: demo-ekscluster
- Region: ap-south-1