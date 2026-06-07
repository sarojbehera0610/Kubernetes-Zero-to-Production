# Project — 3-Tier App Simple Deploy on Kubernetes 🚀

End-to-end deployment of the **EASY_CRUD** 3-tier application on AWS EKS using
plain Kubernetes Pods and Services — no Deployments, no StatefulSets, no Ingress.
This is the simplest possible form of Kubernetes 3-tier deployment, focused on
understanding inter-pod communication and custom image building.

> **Reference Repo**: [abhilashmwaghmare/EASY_CRUD](https://github.com/abhilashmwaghmare/EASY_CRUD/tree/devops)
> Branch: `devops` → folder: `simple-deploy/`

---

## 🏛️ Application Architecture

```
Internet
    │
    ▼
AWS ELB (LoadBalancer)
    │  port 80
    ▼
┌─────────────────┐
│  frontend-pod   │  React app (Nginx)
│  frontend-svc   │  type: LoadBalancer
└────────┬────────┘
         │ http://<NODE_IP>:<NODEPORT>/api
         ▼
┌─────────────────┐
│  backend-pod    │  Spring Boot REST API
│  backend-svc    │  type: NodePort
└────────┬────────┘
         │ jdbc:mariadb://db-svc:3306/student_db
         ▼
┌─────────────────┐
│    db-pod       │  MariaDB
│    db-svc       │  type: ClusterIP
└─────────────────┘
```

### Component Summary

| Tier | Pod Name | Image | Service | Port |
|---|---|---|---|---|
| Database | `db-pod` | `mariadb:latest` | `db-svc` (ClusterIP) | 3306 |
| Backend | `backend-pod` | `sarojbehera/backend:latest` | `backend-svc` (NodePort) | 8080 |
| Frontend | `frontend-app` | `sarojbehera/frontend:latest` | `frontend-svc` (LoadBalancer) | 80 |

---

## 📁 Manifest Files

| File | Creates |
|---|---|
| `db-pod.yaml` | MariaDB Pod + ClusterIP Service (`db-svc`) |
| `backend-pod.yaml` | Spring Boot Pod + NodePort Service (`backend-svc`) |
| `frontend-pod.yaml` | React/Nginx Pod + LoadBalancer Service (`frontend-svc`) |

---

## 🚀 Step-by-Step Practical

---

### 🗄️ PHASE 1 — Deploy the Database

#### Step 1 — Write the DB Pod manifest

```bash
vim db-pod.yaml
```

Paste the following (already in `db-pod.yaml`):

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: db-pod
  labels:
    app: easy-crud-db
spec:
  containers:
    - name: mysql-container
      image: mariadb:latest
      ports:
        - containerPort: 3306
      env:
        - name: MYSQL_ROOT_PASSWORD
          value: cloudblitz
        - name: MYSQL_DATABASE
          value: student_db
---
apiVersion: v1
kind: Service
metadata:
  name: db-svc
spec:
  selector:
    app: easy-crud-db
  ports:
    - port: 3306
      targetPort: 3306
      protocol: TCP
      name: mysql
```

> Key fields to note:
> - `MYSQL_ROOT_PASSWORD: cloudblitz` — this password must match `application.properties` in the backend
> - `MYSQL_DATABASE: student_db` — database name must match the JDBC URL in backend
> - Service name `db-svc` — backend will resolve the DB host using this DNS name inside the cluster

#### Step 2 — Apply the DB pod

```bash
kubectl apply -f db-pod.yaml
```

Verify:

```bash
kubectl get pod db-pod
# NAME     READY   STATUS    RESTARTS
# db-pod   1/1     Running   0

kubectl get svc db-svc
# NAME     TYPE        CLUSTER-IP      PORT(S)
# db-svc   ClusterIP   10.100.xx.xx    3306/TCP
```

Wait for `db-pod` to show `Running` before moving to backend.

#### Step 3 — Verify DB is initialised

```bash
kubectl exec -it db-pod -- mariadb -u root -pcloudblitz -e "SHOW DATABASES;"
# +--------------------+
# | Database           |
# +--------------------+
# | student_db         |
# | information_schema |
# +--------------------+
```

---

### ⚙️ PHASE 2 — Build and Deploy the Backend

The original repo uses `abhilash0104/easy-backend` image. We build our own image
from the same source code after updating the DB config to point to our `db-svc`.

#### Step 4 — Clone the EASY_CRUD repo (on EC2 or local)

```bash
git clone https://github.com/abhilashmwaghmare/EASY_CRUD.git
cd EASY_CRUD
git checkout devops
cd backend
```

#### Step 5 — Update `application.properties`

```bash
vim src/main/resources/application.properties
```

Update the datasource config to match our Kubernetes DB service:

```properties
# Before (original — pointing to localhost or Docker service)
spring.datasource.url=jdbc:mariadb://localhost:3306/student_db

# After — use Kubernetes service DNS name
spring.datasource.url=jdbc:mariadb://db-svc:3306/student_db
spring.datasource.username=root
spring.datasource.password=cloudblitz
spring.datasource.driver-class-name=org.mariadb.jdbc.Driver

spring.jpa.hibernate.ddl-auto=update
spring.jpa.show-sql=true
```

> `db-svc` is the ClusterIP Service name we created in Phase 1. Kubernetes DNS
> resolves `db-svc` to the ClusterIP of the database service inside the cluster.

#### Step 6 — Add Spring Boot Maven plugin to `pom.xml` (if missing)

When we ran `docker build` the first time, the JAR build failed because the
`spring-boot-maven-plugin` was missing from `pom.xml`. Check and add if needed:

```bash
vim pom.xml
```

Ensure this block exists inside `<build><plugins>`:

```xml
<plugin>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-maven-plugin</artifactId>
</plugin>
```

Without this plugin, `mvn package` produces a plain JAR without the embedded
Tomcat — the container starts but the app exits immediately with no error.

#### Step 7 — Build Docker image and push to Docker Hub

```bash
# Make sure you are in the backend/ directory
docker build -t sarojbehera/backend:latest .

# Login to Docker Hub
docker login

# Push image
docker push sarojbehera/backend:latest
```

Verify on Docker Hub: `https://hub.docker.com/r/sarojbehera/backend`

#### Step 8 — Write the Backend Pod manifest

```bash
cd <your-k8s-manifests-folder>
vim backend-pod.yaml
```

The manifest uses `sarojbehera/backend:latest` (our freshly pushed image):

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: backend-pod
  labels:
    app: backend-app
spec:
  containers:
    - name: backend-app
      image: sarojbehera/backend:latest
      ports:
        - containerPort: 8080
---
apiVersion: v1
kind: Service
metadata:
  name: backend-svc
spec:
  selector:
    app: backend-app
  ports:
    - port: 8080
      targetPort: 8080
      protocol: TCP
      name: java
  type: NodePort
```

#### Step 9 — Apply the Backend pod

```bash
kubectl apply -f backend-pod.yaml
```

Verify:

```bash
kubectl get pod backend-pod
# NAME          READY   STATUS    RESTARTS
# backend-pod   1/1     Running   0

kubectl get svc backend-svc
# NAME          TYPE       CLUSTER-IP      PORT(S)
# backend-svc   NodePort   10.100.xx.xx    8080:3xxxx/TCP
```

Note the assigned NodePort (e.g. `32456`) — you will need it for the frontend
`.env` update.

#### Step 10 — Check backend logs for DB connection

```bash
kubectl logs backend-pod --tail=30
# ...HikariPool-1 - Start completed.
# ...Started EasyCrudApplication in 9.2 seconds (process running for 10.1)
```

If you see `HikariPool` start completed — backend has successfully connected to
`db-svc:3306`.

---

### 🌐 PHASE 3 — Build and Deploy the Frontend

#### Step 11 — Get the Node external IP and Backend NodePort

```bash
# Node external IP
kubectl get nodes -o wide
# EXTERNAL-IP column → e.g. 13.xx.xx.xx

# Backend NodePort
kubectl get svc backend-svc
# 8080:32456/TCP → NodePort is 32456
```

#### Step 12 — Update the Frontend `.env` file

```bash
cd EASY_CRUD/frontend
vim .env
```

Update the API base URL to point to the backend via NodePort:

```env
# Before (original — localhost or Docker compose internal)
REACT_APP_API_URL=http://localhost:8080

# After — Node external IP + backend NodePort
REACT_APP_API_URL=http://13.xx.xx.xx:32456
```

> This is baked into the React build at `docker build` time. If the IP/port
> changes, you must rebuild and repush the image.

#### Step 13 — Build Docker image and push to Docker Hub

```bash
# In EASY_CRUD/frontend/
docker build -t sarojbehera/frontend:latest .
docker push sarojbehera/frontend:latest
```

#### Step 14 — Write the Frontend Pod manifest

```bash
vim frontend-pod.yaml
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: frontend-app
  labels:
    app: frontend-app
spec:
  containers:
    - name: frontend-app
      image: sarojbehera/frontend:latest
      ports:
        - containerPort: 80
---
kind: Service
apiVersion: v1
metadata:
  name: frontend-svc
spec:
  ports:
    - name: http
      protocol: TCP
      port: 80
      targetPort: 80
  selector:
    app: frontend-app
  type: LoadBalancer
```

#### Step 15 — Apply the Frontend pod

```bash
kubectl apply -f frontend-pod.yaml
```

Verify:

```bash
kubectl get pod frontend-app
# NAME           READY   STATUS    RESTARTS
# frontend-app   1/1     Running   0

kubectl get svc frontend-svc
# NAME           TYPE           CLUSTER-IP      EXTERNAL-IP
# frontend-svc   LoadBalancer   10.100.xx.xx    <ELB-DNS>.elb.amazonaws.com
```

Wait ~2 minutes for the ELB DNS to populate in `EXTERNAL-IP`.

---

### ✅ PHASE 4 — Test the Application

#### Step 16 — Open the App in Browser

```
http://<ELB-DNS>.ap-south-1.elb.amazonaws.com
```

You should see the EASY_CRUD student registration form.

#### Step 17 — Verify Data Saving

1. Enter a student name and email in the form
2. Click **Add Student**
3. The student appears in the list below the form
4. Verify it persisted in the DB:

```bash
kubectl exec -it db-pod -- mariadb -u root -pcloudblitz student_db \
  -e "SELECT * FROM student;"
# +----+--------+-------------------+
# | id | name   | email             |
# +----+--------+-------------------+
# |  1 | Saroj  | saroj@test.com    |
# +----+--------+-------------------+
```

---

## 🐛 Errors Faced & Troubleshooting

### ❌ Error 1 — Backend pod `CrashLoopBackOff` — Spring Boot exits immediately

**Symptom:**
```bash
kubectl get pod backend-pod
# NAME          READY   STATUS             RESTARTS
# backend-pod   0/1     CrashLoopBackOff   3

kubectl logs backend-pod
# no main manifest attribute, in /app/app.jar
```

**Cause:** `spring-boot-maven-plugin` was missing from `pom.xml`. Maven built a
plain JAR (without embedded Tomcat) so the container had nothing to run.

**Fix:** Add the plugin to `pom.xml` → rebuild image → repush → delete old pod
and re-apply:

```xml
<build>
  <plugins>
    <plugin>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-maven-plugin</artifactId>
    </plugin>
  </plugins>
</build>
```

```bash
# Rebuild after pom.xml fix
docker build -t sarojbehera/backend:latest .
docker push sarojbehera/backend:latest

# Delete old pod (pods are immutable — must delete and recreate)
kubectl delete pod backend-pod
kubectl apply -f backend-pod.yaml
```

---

### ❌ Error 2 — Backend connects but DB tables not created

**Symptom:** Backend started successfully but API returned `Table 'student_db.student'
doesn't exist`.

**Cause:** `spring.jpa.hibernate.ddl-auto` was not set or was set to `none`.
Hibernate did not auto-create the table on first boot.

**Fix:** Set in `application.properties`:
```properties
spring.jpa.hibernate.ddl-auto=update
```

Rebuild and redeploy backend. On next startup, Hibernate creates the `student`
table automatically.

---

### ❌ Error 3 — Frontend loads but student list empty / API calls fail

**Symptom:** Form appeared but adding a student showed an error. Browser console:
```
Failed to fetch http://localhost:8080/api/students
```

**Cause:** `.env` was not updated before building the Docker image. The original
`localhost:8080` was baked into the React bundle.

**Fix:** Update `.env`, rebuild, repush, delete and recreate frontend pod:
```bash
# Update .env
REACT_APP_API_URL=http://<NODE_IP>:<BACKEND_NODEPORT>

# Rebuild
docker build -t sarojbehera/frontend:latest .
docker push sarojbehera/frontend:latest

# Delete old pod
kubectl delete pod frontend-app
kubectl apply -f frontend-pod.yaml
```

---

### ❌ Error 4 — `ImagePullBackOff` on backend or frontend pod

**Symptom:**
```bash
kubectl describe pod backend-pod
# Failed to pull image "sarojbehera/backend:latest": not found
```

**Cause:** Image was not pushed to Docker Hub before applying the manifest, or
the tag name had a typo.

**Fix:**
```bash
# Check the image exists on Docker Hub
docker pull sarojbehera/backend:latest

# If pull fails — push first
docker push sarojbehera/backend:latest

# Force pod to re-pull
kubectl delete pod backend-pod
kubectl apply -f backend-pod.yaml
```

---

### ❌ Error 5 — `frontend-svc` EXTERNAL-IP stuck at `<pending>`

**Symptom:**
```bash
kubectl get svc frontend-svc
# NAME           TYPE           CLUSTER-IP   EXTERNAL-IP   PORT(S)
# frontend-svc   LoadBalancer   10.100.x.x   <pending>     80:3xxxx/TCP
```

**Cause:** EKS could not provision the ELB because the node subnets were missing
the required tag.

**Fix:** Tag the public subnets in ap-south-1:
```bash
aws ec2 create-tags \
  --resources <subnet-id> \
  --tags Key=kubernetes.io/role/elb,Value=1
```

After tagging, ELB DNS populated within 2–3 minutes.

---

## 📌 Key Takeaways

- **Deploy order matters** — DB must be `Running` before backend, backend must be
  `Running` before building the frontend image (you need the NodePort)
- **`db-svc`** is how the backend finds the DB inside the cluster — Kubernetes DNS
  resolves service names automatically
- **`.env` is baked at build time** in React — always update it before `docker build`
- **Pods are immutable** — after changing an image, you must `kubectl delete pod`
  and `kubectl apply` again (unlike Deployments which roll automatically)
- **`spring-boot-maven-plugin`** must be in `pom.xml` or the JAR has no main manifest
- **NodePort** is dynamically assigned — check `kubectl get svc backend-svc` each
  time and update `.env` accordingly

---

## ⚡ Quick Commands Reference

```bash
# Full deploy sequence
kubectl apply -f db-pod.yaml
kubectl apply -f backend-pod.yaml
kubectl apply -f frontend-pod.yaml

# Check all pods
kubectl get pods
kubectl get svc

# DB shell
kubectl exec -it db-pod -- mariadb -u root -pcloudblitz

# Backend logs
kubectl logs backend-pod --tail=30

# Get Node IP
kubectl get nodes -o wide

# Get backend NodePort
kubectl get svc backend-svc

# Get frontend ELB DNS
kubectl get svc frontend-svc

# Delete and recreate a pod (after image update)
kubectl delete pod backend-pod && kubectl apply -f backend-pod.yaml
kubectl delete pod frontend-app && kubectl apply -f frontend-pod.yaml

# Full cleanup
kubectl delete -f frontend-pod.yaml
kubectl delete -f backend-pod.yaml
kubectl delete -f db-pod.yaml
```

---

> **Author**: Saroj Behera | [sarojops.cloud](https://sarojops.cloud) | [GitHub](https://github.com/sarojbehera0610)