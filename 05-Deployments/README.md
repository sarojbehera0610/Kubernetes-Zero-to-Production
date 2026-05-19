# Deployments

A Deployment manages ReplicaSets and provides declarative updates,
rollouts, and rollbacks for stateless applications.

## Deployment Manifest
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-deploy
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-app
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 1
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
        - name: nginx
          image: nginx:1.19
          ports:
            - containerPort: 80
```

## Deployment Strategies

### Recreate
```yaml
strategy:
  type: Recreate
```
- All old pods terminated first, then new pods created
- Downtime occurs during transition
- Use for non-critical updates

### Rolling Update (default)
```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxSurge: 1        # max extra pods during update
    maxUnavailable: 1  # max pods that can be down
```
- Pods replaced one by one
- Zero downtime
- Default strategy

## Rollout Commands
```bash
# trigger rollout by updating image
kubectl set image deploy/my-deploy nginx=nginx:1.21

# watch rollout progress
kubectl rollout status deploy my-deploy

# view rollout history
kubectl rollout history deploy my-deploy

# rollback to previous version
kubectl rollout undo deploy my-deploy

# rollback to specific revision
kubectl rollout undo deploy my-deploy --to-revision=2
```

## Key Learning
- Deployment = ReplicaSet + rollout/rollback capability
- `kubectl rollout status` streams live pod replacement progress
- Each image change creates a new ReplicaSet — old one is kept for rollback