# Kubernetes ConfigMap and Secrets

## What is a ConfigMap

A ConfigMap stores non-sensitive configuration data as key-value pairs. It decouples configuration from the container image so you do not need to rebuild images when config changes. Examples include environment variables, config files, command line arguments.

## What is a Secret

A Secret stores sensitive data like passwords, tokens, and keys. Data is base64 encoded (not encrypted by default). Should be used instead of hardcoding passwords in manifests.

---

## ConfigMap vs Secret

ConfigMap - plain text, non-sensitive config, readable by anyone with cluster access.

Secret - base64 encoded, sensitive data, can be encrypted at rest with additional setup.

---

## ConfigMap Example

### configmap.yaml
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: my-config
data:
  APP_ENV: production
  APP_PORT: "8080"
  APP_NAME: saroj-app
```

### Using ConfigMap in Pod as environment variables
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
spec:
  containers:
    - name: container-1
      image: nginx
      envFrom:
        - configMapRef:
            name: my-config
```

### Using specific keys from ConfigMap
```yaml
env:
  - name: ENVIRONMENT
    valueFrom:
      configMapKeyRef:
        name: my-config
        key: APP_ENV
```

---

## Secret Example

### Create base64 encoded values
```bash
echo -n "saroj" | base64
# c2Fyb2o=

echo -n "saroj123" | base64
# c2Fyb2oxMjM=
```

### secret.yaml
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: my-secret
type: Opaque
data:
  MYSQL_ROOT_USER: c2Fyb2o=
  MYSQL_ROOT_PASSWORD: c2Fyb2oxMjM=
```

### Using Secret in Pod
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
spec:
  containers:
    - name: container-1
      image: mysql:8
      envFrom:
        - secretRef:
            name: my-secret
```

### Using specific keys from Secret
```yaml
env:
  - name: MYSQL_ROOT_PASSWORD
    valueFrom:
      secretKeyRef:
        name: my-secret
        key: MYSQL_ROOT_PASSWORD
```

---

## Apply and Verify
```bash
kubectl apply -f configmap.yaml
kubectl apply -f secret.yaml

kubectl get configmap
kubectl get secret

kubectl describe configmap my-config
kubectl describe secret my-secret
# Secret values are hidden in describe output

# Decode a secret value
kubectl get secret my-secret -o jsonpath="{.data.MYSQL_ROOT_PASSWORD}" | base64 --decode

# Verify env variables inside pod
kubectl exec -it my-pod -- env | grep MYSQL
kubectl exec -it my-pod -- env | grep APP
```

---

## Why Use ConfigMap and Secret Instead of Hardcoding

Hardcoding passwords in manifest files is a security risk. If the file is pushed to GitHub the password is exposed. Using Secrets keeps sensitive values separate from the manifest. ConfigMaps allow config changes without rebuilding Docker images. Both can be updated independently without touching the Pod spec.