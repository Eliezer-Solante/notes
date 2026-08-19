
![[Pasted image 20260813163220.png]]
![[Pasted image 20260813163235.png]]

To display all object that was created
![[Pasted image 20260813163250.png]]



A **Deployment** is the controller you actually use in practice for stateless applications. It manages ReplicaSets for you, and adds the one capability neither RC nor RS have on their own: **rolling updates and rollbacks**.

```
Deployment → manages → ReplicaSet → manages → Pods
```

## Example YAML

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
  labels:
    app: my-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-app
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
        - name: app
          image: my-app:1.0
          ports:
            - containerPort: 8080
```

- **`strategy`** — this is the field RC/RS don't have. `RollingUpdate` replaces pods gradually; `maxSurge` controls how many extra pods can be created above the desired count during the rollout, `maxUnavailable` controls how many can be down at once. The alternative is `type: Recreate`, which kills all old pods before creating new ones (downtime, but simpler).

## Common commands

```bash
# create/apply
kubectl apply -f deployment.yaml

# list deployments
kubectl get deployments
kubectl get deploy -o wide

# describe (shows rollout status, conditions, events)
kubectl describe deployment my-app

# scale
kubectl scale deployment my-app --replicas=5

# edit live
kubectl edit deployment my-app

# update the image directly (triggers a rolling update)
kubectl set image deployment/my-app app=my-app:2.0

# watch the rollout happen in real time
kubectl rollout status deployment/my-app

# see rollout history
kubectl rollout history deployment/my-app

# roll back to the previous version
kubectl rollout undo deployment/my-app

# roll back to a specific revision
kubectl rollout undo deployment/my-app --to-revision=2

# pause/resume a rollout (useful for canary-style batching of changes)
kubectl rollout pause deployment/my-app
kubectl rollout resume deployment/my-app

# delete
kubectl delete deployment my-app
```

## Why this matters relative to what we've covered

When you run `kubectl set image` or edit the image tag, the Deployment doesn't modify pods in place — it creates a **new ReplicaSet** with the updated template, scales it up, and scales the old ReplicaSet down to zero, one pod at a time based on your `strategy`. That's the mechanism behind `kubectl get rs` showing two ReplicaSets during a rollout, which we touched on earlier.

## Quick full recap of the chain

|kind|manages|adds|
|---|---|---|
|ReplicationController|Pods|basic self-healing (legacy)|
|ReplicaSet|Pods|set-based selectors (modern low-level)|
|Deployment|ReplicaSets|rolling updates, rollback, pause/resume|

In everyday work: you write Deployment YAML, `kubectl apply` it, and let it handle the ReplicaSet and Pod layers underneath automatically.

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"apps/v1","kind":"ReplicaSet","metadata":{"annotations":{},"labels":{"app":"catalog"},"name":"catalog-replicas","namespace":"catalog-ops"},"spec":{"replicas":3,"selector":{"matchLabels":{"app":"catalog"}},"template":{"metadata":{"labels":{"app":"catalog"}},"spec":{"containers":[{"image":"nginx:1.25-alpine","name":"catalog"}]}}}}
  creationTimestamp: "2026-08-19T02:03:09Z"
  generation: 1
  labels:
    app: catalog
  name: catalog-replicas
  namespace: catalog-ops
  resourceVersion: "1232"
  uid: 257c6ce5-802c-41ea-a6ac-5ef5d41b156d
spec:
  replicas: 3
  selector:
    matchLabels:
      app: catalog
  template:
    metadata:
      labels:
        app: catalog
    spec:
      containers:
      - image: nginx:1.25-alpine
        imagePullPolicy: IfNotPresent
        name: catalog
        resources: {}
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: File
      dnsPolicy: ClusterFirst
      restartPolicy: Always
      schedulerName: default-scheduler
      securityContext: {}
      terminationGracePeriodSeconds: 30
status:
  availableReplicas: 3
  fullyLabeledReplicas: 3
  observedGeneration: 1
  readyReplicas: 3
  replicas: 3
  terminatingReplicas: 0
```