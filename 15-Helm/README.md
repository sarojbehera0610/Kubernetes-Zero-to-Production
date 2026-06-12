# 15 — Helm: Kubernetes Package Manager

## What is Helm?
Helm is the package manager for Kubernetes — exactly like `apt` for Ubuntu, `pip` for Python, or `npm` for Node.js. It bundles all the Kubernetes YAMLs needed for an application into a single unit called a **Chart**, and manages the full lifecycle of that application on your cluster.

Without Helm, installing a complex application like KEDA requires manually applying 20+ YAML files in the correct order. With Helm, it's one command.

## Core Concepts

### Chart
A pre-packaged bundle of all Kubernetes YAMLs for one application. Someone already wrote all the manifests — you just install them.
kedacore/keda        → Chart for KEDA
nginx-stable/nginx-ingress  → Chart for NGINX Ingress Controller
oci://public.ecr.aws/karpenter/karpenter → Chart for Karpenter

### Repository
A hosted collection of Charts — like an app store.
kedacore     → maintained by KEDA team
nginx-stable → maintained by NGINX
bitnami      → 100+ popular apps

### Release
A named instance of a Chart installed in your cluster. You can install the same Chart multiple times with different release names.
helm install keda kedacore/keda
↑         ↑
release name  chart

## How Helm Works Internally
helm install keda kedacore/keda --namespace keda
│
▼

Downloads Chart from kedacore repo
│
▼
Renders all YAML templates inside the Chart
│
▼
Applies all YAMLs to cluster (like kubectl apply)
│
▼
Saves release record as a Secret in cluster
│
▼
Result: Application running in your cluster


## Installation
```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
helm version
# version.BuildInfo{Version:"v3.21.0", ...}
```

Helm reads `~/.kube/config` automatically — same kubeconfig as kubectl. No extra setup needed.

## Essential Commands

### Repository Management
```bash
helm repo add <name> <url>      # add a chart repository
helm repo update                # refresh repo catalog (like apt update)
helm repo list                  # show all added repos
helm repo remove <name>         # remove a repo
```

### Searching
```bash
helm search repo <keyword>      # search in added repos
helm search hub <keyword>       # search Artifact Hub (all public charts)
```

### Installing and Managing
```bash
helm install <release> <chart> --namespace <ns>        # install
helm upgrade --install <release> <chart> --namespace <ns>  # install or upgrade
helm list                       # releases in default namespace
helm list -n <namespace>        # releases in specific namespace
helm list -A                    # all releases across all namespaces
helm uninstall <release> -n <ns>  # remove a release
```

### Inspecting
```bash
helm status <release> -n <ns>   # health of a release
helm history <release> -n <ns>  # version history and revisions
helm get values <release> -n <ns>  # see configured values
```

## Helm vs kubectl

| | kubectl | Helm |
|---|---|---|
| Applies YAMLs | ✅ | ✅ |
| Tracks what was installed | ❌ | ✅ |
| Rollback to previous version | ❌ | ✅ |
| Upgrade with one command | ❌ | ✅ |
| Parameterized installs (--set) | ❌ | ✅ |
| Uninstall everything at once | ❌ | ✅ |

## Helm Used in This Repository

### 1. NGINX Ingress Controller (Exploration)
Used to learn and explore Helm basics — adding repos, searching charts, installing and uninstalling releases.

```bash
# Add nginx repo
helm repo add nginx-stable https://helm.nginx.com/stable
helm repo update

# Search available charts
helm search repo nginx

# Install nginx ingress controller
helm install my-nginx nginx-stable/nginx-ingress

# Verify — Helm created Deployment, Service, ServiceAccount automatically
helm list
kubectl get pods
kubectl get svc

# Uninstall — removes everything Helm created
helm uninstall my-nginx
```

**What Helm created automatically:**
- Deployment → nginx ingress controller pod
- Service (LoadBalancer) → AWS ELB provisioned
- ServiceAccount, ClusterRole, ClusterRoleBinding
- ConfigMap

**Key learning:** One `helm install` replaced ~8 separate `kubectl apply` commands.

### 2. KEDA (Kubernetes Event-Driven Autoscaling)
Installed KEDA into its own namespace using the official kedacore Helm chart.

```bash
helm repo add kedacore https://kedacore.github.io/charts
helm repo update
kubectl create namespace keda
helm install keda kedacore/keda --namespace keda
```

**What Helm created:**
- keda-operator Deployment
- keda-metrics-apiserver Deployment  
- keda-admission-webhooks Deployment
- All associated Services, ServiceAccounts, ClusterRoles, CRDs

**Helm flags used:**
- `--namespace keda` → install into specific namespace

### 3. Karpenter (Node Provisioner)
Installed Karpenter from AWS ECR public registry using OCI format chart with multiple `--set` overrides.

```bash
helm upgrade --install karpenter oci://public.ecr.aws/karpenter/karpenter \
  --version 1.3.3 \
  --namespace kube-system \
  --set settings.clusterName=ekscluster \
  --set settings.interruptionQueue=ekscluster \
  --set controller.resources.requests.cpu=100m \
  --set controller.resources.requests.memory=256Mi \
  --set replicas=1 \
  --set serviceAccount.annotations."eks\.amazonaws\.com/role-arn"="arn:aws:iam::051997448832:role/KarpenterControllerRole-ekscluster"
```

**What Helm created:**
- Karpenter controller Deployment
- ServiceAccount with IRSA annotation
- ClusterRole and ClusterRoleBinding
- Webhook configurations
- CRDs: NodePool, EC2NodeClass, NodeClaim

**Helm flags used:**
- `--version` → pin specific chart version
- `--upgrade --install` → install if not exists, upgrade if exists
- `--set` → override chart default values at install time
- OCI registry format (`oci://`) → chart hosted on AWS ECR

## Key Learnings

1. **`helm upgrade --install` is safer than `helm install`** — it won't fail if the release already exists. Always use this in scripts and CI/CD pipelines.

2. **`--set` vs values file** — for a few overrides use `--set`. For many overrides create a `values.yaml` file and use `-f values.yaml`.

3. **Helm stores state as Secrets** — run `kubectl get secrets -n kube-system | grep helm` to see release records. This is how Helm tracks what it installed for upgrades and rollbacks.

4. **OCI registries** — newer charts are hosted on OCI registries (like AWS ECR) instead of traditional HTTP repos. Use `oci://` prefix instead of `helm repo add`.

5. **Namespace isolation** — always install charts into dedicated namespaces (`--namespace keda`, `--namespace kube-system`). Makes management and cleanup easier.

6. **`helm repo update` before install** — always run this before installing, otherwise you may get outdated chart versions.

## Helm Release Lifecycle
helm install    → REVISION 1 (deployed)
helm upgrade    → REVISION 2 (deployed)
helm upgrade    → REVISION 3 (deployed)
helm rollback   → REVISION 4 (rolls back to previous)
helm uninstall  → release deleted, all K8s objects removed