# ReplicationController vs ReplicaSet

Both ensure a specified number of Pod replicas are always running.

## ReplicationController (legacy)
```yaml
apiVersion: v1
kind: ReplicationController
metadata:
  name: my-rc
spec:
  replicas: 3
  selector:
    app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
        - name: my-container
          image: nginx:latest
```

## ReplicaSet (modern)
```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: my-rs
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
        - name: my-container
          image: nginx:latest
```

## Key Difference

| Feature | ReplicationController | ReplicaSet |
|---|---|---|
| Selector type | Equality only (`app: my-app`) | Set-based (`matchLabels`, `matchExpressions`) |
| Usage | Legacy, being phased out | Used by Deployments internally |
| Flexibility | Limited | High |

## Key Learning
- Never use RC in production — use ReplicaSet or Deployment
- Deployment manages ReplicaSet automatically under the hood
- ReplicaSet alone has no rollout/rollback — use Deployment for that