# 11 — Ingress (Path-Based Routing with NGINX) 🌐

Hands-on Ingress implementation on AWS EKS using the NGINX Ingress Controller.
Deployed a frontend (React) and backend (Spring Boot) as separate Deployments and
exposed them both under a **single LoadBalancer URL** using path-based routing.

---

## 📖 Concept

### What is Ingress?

Without Ingress, exposing multiple services externally means creating one
LoadBalancer Service per service — each provisioning its own AWS ELB, costing
money and adding complexity.

**Ingress** is a Kubernetes resource that acts as a smart Layer-7 HTTP router
sitting in front of your services. A single LoadBalancer (NLB/ALB) receives all
traffic, and the Ingress Controller routes requests to the correct backend Service
based on **host** or **path** rules.

```
Internet
    │
    ▼
AWS NLB (single IP/DNS)
    │
    ▼
NGINX Ingress Controller Pod
    │
    ├── /api/*   →  backend-service:8080  (Spring Boot)
    │
    └── /*       →  frontend-service:80   (React/Nginx)
```

### Ingress vs Service LoadBalancer

| Feature | Service (LoadBalancer) | Ingress |
|---|---|---|
| Layer | L4 (TCP) | L7 (HTTP/HTTPS) |
| AWS resource | 1 ELB per service | 1 NLB for all services |
| Path routing | ❌ | ✅ |
| Host routing | ❌ | ✅ |
| SSL termination | Manual | ✅ via TLS block |
| Cost | High (multiple ELBs) | Low (single NLB) |

### Key Annotations Used

| Annotation | Purpose |
|---|---|
| `nginx.ingress.kubernetes.io/rewrite-target: /$2` | Strips the matched path prefix before forwarding to backend |
| `nginx.ingress.kubernetes.io/use-regex: "true"` | Enables regex patterns in path definitions |

### `pathType: ImplementationSpecific`

Used instead of `Exact` or `Prefix` because regex rewrite rules require the NGINX
controller to interpret the path — standard Kubernetes path types do not support
capture groups like `(/|$)(.*)`.

---

## 📋 Prerequisites

### Install NGINX Ingress Controller on EKS

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.9.4/deploy/static/provider/aws/deploy.yaml
```

Verify the controller pod is running:

```bash
kubectl get pods -n ingress-nginx
# NAME                                        READY   STATUS
# ingress-nginx-controller-xxxxxxxxx-xxxxx    1/1     Running

kubectl get svc -n ingress-nginx
# NAME                       TYPE           CLUSTER-IP     EXTERNAL-IP
# ingress-nginx-controller   LoadBalancer   10.100.xx.xx   <NLB-DNS>.elb.amazonaws.com
```

> The `EXTERNAL-IP` is your NLB DNS — all traffic enters here. Copy this — you
> will use it to test both frontend and backend.

---

## 📁 Manifest Files

| File | Purpose |
|---|---|
| `frontend-deployment-service.yaml` | React frontend Deployment (2 replicas) + ClusterIP Service |
| `backend-deployment-service.yaml` | Spring Boot backend Deployment (2 replicas) + ClusterIP Service |
| `ingress.yaml` | Ingress resource with path-based rules — `/api/*` → backend, `/*` → frontend |

---

## 🚀 Step-by-Step Practical

### Step 1 — Install NGINX Ingress Controller

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.9.4/deploy/static/provider/aws/deploy.yaml

# Wait for controller to be ready
kubectl rollout status deployment/ingress-nginx-controller -n ingress-nginx
```

### Step 2 — Apply Frontend Deployment and Service

```bash
kubectl apply -f frontend-deployment-service.yaml
kubectl get pods -l app=frontend
# NAME                                   READY   STATUS
# frontend-deployment-xxxxxxxxx-aaa      1/1     Running
# frontend-deployment-xxxxxxxxx-bbb      1/1     Running

kubectl get svc frontend-service
# NAME               TYPE        CLUSTER-IP      PORT(S)
# frontend-service   ClusterIP   10.100.xx.xx    80/TCP
```

### Step 3 — Apply Backend Deployment and Service

```bash
kubectl apply -f backend-deployment-service.yaml
kubectl get pods -l app=backend
kubectl get svc backend-service
```

### Step 4 — Apply Ingress Resource

```bash
kubectl apply -f ingress.yaml
kubectl get ingress
# NAME               CLASS   HOSTS   ADDRESS                              PORTS
# easycrud-ingress   nginx   *       <NLB-DNS>.ap-south-1.elb.amazonaws.com   80
```

> The `ADDRESS` field populates within ~2 minutes as AWS provisions the NLB.

### Step 5 — Test Path-Based Routing

```bash
NLB_DNS=$(kubectl get svc -n ingress-nginx ingress-nginx-controller \
  -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')

echo "NLB: $NLB_DNS"

# Test frontend — should return React HTML
curl http://$NLB_DNS/
# <!DOCTYPE html><html>...React App...

# Test backend API — should return JSON student list
curl http://$NLB_DNS/api/students
# [{"id":1,"name":"Saroj","email":"saroj@test.com"}, ...]
```

---

## 🐛 Errors Faced & Troubleshooting

### ❌ Error 1 — Ingress ADDRESS stays empty

**Symptom:**
```bash
kubectl get ingress
# NAME               CLASS   HOSTS   ADDRESS   PORTS
# easycrud-ingress   nginx   *                 80
```

**Cause:** NGINX controller's LoadBalancer service could not provision the NLB
because the EKS node subnets were not tagged for public load balancers.

**Fix:** Tag the public subnets in ap-south-1:
```bash
aws ec2 create-tags \
  --resources subnet-xxxxxxxx \
  --tags Key=kubernetes.io/role/elb,Value=1
```

After tagging, the NLB provisioned within 2–3 minutes.

---

### ❌ Error 2 — `/api/*` requests returning 404

**Symptom:** Frontend loaded fine but `curl http://$NLB_DNS/api/students` returned
404 from Spring Boot (not from Ingress).

**Cause:** The `rewrite-target: /$2` annotation was stripping the `/api` prefix
correctly, but Spring Boot's context path still expected `/api/students`. The
rewrite was sending `/students` to the app which had no such mapping.

**Fix:** Remove the `rewrite-target` annotation and keep paths as-is:
```yaml
# ingress.yaml — removed rewrite-target
# path: /api
# pathType: Prefix
```
Spring Boot already handled `/api/students` — the rewrite was the problem,
not the solution. Simplifying to `Prefix` path type resolved it.

---

### ❌ Error 3 — `404 nginx` on all paths (not reaching any service)

**Symptom:** Every request returned the default NGINX 404 page.

**Cause:** `ingressClassName: nginx` was missing from the Ingress spec. The NGINX
controller ignored the Ingress resource because it was not assigned to it.

**Fix:**
```yaml
spec:
  ingressClassName: nginx    # ← this line was missing
```

---

### ❌ Error 4 — Frontend loaded but showed blank page (CORS / API error)

**Symptom:** React app loaded but student list was empty, browser console showed
`CORS policy blocked` on `/api/students`.

**Cause:** The React app was built with `REACT_APP_API_URL=http://localhost:8080`.
In a Kubernetes setup, the API URL must be the same NLB DNS under `/api`.

**Fix:** Rebuild the Docker image with the correct API base URL:
```bash
docker build \
  --build-arg REACT_APP_API_URL=http://<NLB_DNS>/api \
  -t sarojbehera/easycrud-frontend:ingress .
docker push sarojbehera/easycrud-frontend:ingress
```
Update the image tag in `frontend-deployment-service.yaml` and re-apply.

---

## 📌 Key Takeaways

- Ingress = **one NLB, many services** — cost-efficient on AWS
- Always set `ingressClassName: nginx` or controller ignores your resource
- `rewrite-target` with `$2` capture group is powerful but tricky — test with
  `curl -v` to see what URL actually hits your backend
- Subnet tagging (`kubernetes.io/role/elb=1`) is mandatory for EKS NLB provisioning
- Use `pathType: ImplementationSpecific` only when regex rewriting — use `Prefix`
  for simple cases

---

## ⚡ Commands Reference

```bash
# Install NGINX Ingress Controller
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.9.4/deploy/static/provider/aws/deploy.yaml

# Apply manifests
kubectl apply -f frontend-deployment-service.yaml
kubectl apply -f backend-deployment-service.yaml
kubectl apply -f ingress.yaml

# Get Ingress status
kubectl get ingress
kubectl describe ingress easycrud-ingress

# Get NLB DNS
kubectl get svc -n ingress-nginx ingress-nginx-controller

# Test routing
curl http://<NLB-DNS>/
curl http://<NLB-DNS>/api/students

# Logs from NGINX controller
kubectl logs -n ingress-nginx deployment/ingress-nginx-controller

# Cleanup
kubectl delete -f ingress.yaml
kubectl delete -f frontend-deployment-service.yaml
kubectl delete -f backend-deployment-service.yaml
```

---

> **Author**: Saroj Behera | [sarojops.cloud](https://sarojops.cloud) | [GitHub](https://github.com/sarojbehera0610)