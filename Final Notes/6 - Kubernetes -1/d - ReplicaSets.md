
## ==Replication Controller==

A **ReplicationController (RC)** is the original, older version of what a ReplicaSet does — it ensures a specified number of pod replicas are running at all times. It's the predecessor technology; ReplicaSets were introduced later to replace it with a more flexible selector syntax.

## ReplicationController vs ReplicaSet

![[Pasted image 20260813140657.png]]

Kubernetes documentation itself now recommends using Deployments (which manage ReplicaSets) instead of RCs directly — RCs are kept around mostly for backward compatibility with older clusters and tutorials.

## Example YAML

```yaml
apiVersion: v1
kind: ReplicationController
metadata:
  name: my-app-rc
  labels:
    app: my-app
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
        - name: app
          image: my-app:1.0
          ports:
            - containerPort: 8080
```

Notice `selector` here is a flat key-value map (`app: my-app`) rather than the `matchLabels`/`matchExpressions` structure a ReplicaSet uses — that's the main syntactic difference.

## How it's created

```bash
# create from a YAML file
kubectl apply -f rc.yaml
# or the older, equivalent command
kubectl create -f rc.yaml

# verify it's running
kubectl get rc
kubectl get replicationcontroller my-app-rc

# see the pods it's managing
kubectl get pods -l app=my-app

# describe for events and status
kubectl describe rc my-app-rc

# scale it
kubectl scale rc my-app-rc --replicas=5

# delete it (and cascade to its pods by default)
kubectl delete rc my-app-rc
```

Sample:
![[Pasted image 20260813141405.png]]

![[Pasted image 20260813141417.png]]
## Why it still comes up

You'll mostly encounter RCs in older documentation, legacy clusters, or certification study material (like the CKA exam, which still tests on it historically). For anything you build today, the practical chain is what we covered before:

```
Deployment → ReplicaSet → Pods
```

with ReplicationController sitting as the deprecated ancestor of ReplicaSet in that lineage.

---

## ==ReplicaSets==

A **ReplicaSet** is the controller that keeps a specified number of identical pod replicas running at all times. If a pod crashes or a node dies, the ReplicaSet notices the count dropped and creates a replacement.

## How it relates to what we've covered

You almost never write a ReplicaSet directly — a **Deployment** creates and manages a ReplicaSet for you underneath. When you update a Deployment's image version, it actually creates a _new_ ReplicaSet and scales the old one down to zero, which is what enables rolling updates and rollbacks. So the chain is:

```
Deployment → manages → ReplicaSet → manages → Pods
```

ReplicaSets on their own don't support rolling updates — that's the extra layer Deployments add. This is why ReplicaSets are considered a lower-level building block rather than something you interact with day to day.

## Example YAML

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: my-app-rs
  labels:
    app: my-app
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
        - name: app
          image: my-app:1.0
          ports:
            - containerPort: 8080
```

`selector.matchLabels` is the important piece — it tells the ReplicaSet which pods belong to it. If a pod matching that label exists but wasn't created by this ReplicaSet, it can be adopted; if the count is under 3, new ones get created from `template`.

## Common commands

```bash
# apply the manifest
kubectl apply -f replicaset.yaml

# list ReplicaSets
kubectl get rs
kubectl get replicasets -o wide

# see details, including recent scaling events
kubectl describe rs my-app-rs

# check how many pods it currently owns
kubectl get pods -l app=my-app

# scale manually
kubectl scale rs my-app-rs --replicas=5

# delete the ReplicaSet but keep its pods running (orphans them)
kubectl delete rs my-app-rs --cascade=orphan

# delete the ReplicaSet and its pods
kubectl delete rs my-app-rs

# edit live
kubectl edit rs my-app-rs
```

## Why you'd look at ReplicaSets even if you don't manage them directly

When debugging a Deployment rollout, `kubectl get rs` is genuinely useful — you'll see the old and new ReplicaSet side by side during a rolling update, one scaling down as the other scales up:

```bash
kubectl get rs -l app=my-app
# NAME               DESIRED   CURRENT   READY
# my-app-7d9f8c6b5    0         0         0
# my-app-5b6c9d4f8    3         3         3
```

That's the mechanism a rolling update actually uses under the hood — it's not magic, it's just two ReplicaSets handing off pod count gradually.

<mark style="background: #FFB86CA6;">Sample:</mark>
![[Pasted image 20260813161908.png]]
![[Pasted image 20260813161923.png]]

### <mark style="background: #ABF7F7A6;">For editing a replicaset file</mark>
## Method 1: `kubectl scale` (direct, one-liner)

```bash
kubectl scale deployment my-app --replicas=5
```

This works the same way for other kinds too:

```bash
kubectl scale rs my-app-rs --replicas=5
kubectl scale rc my-app-rc --replicas=5
```

Output confirms the change immediately:

```
deployment.apps/my-app scaled
```

## Method 2: `kubectl edit` (manual, opens live YAML in your editor)

```bash
kubectl edit deployment my-app
```

This opens the live object's YAML in your default terminal editor (vim, nano, whatever `$EDITOR` is set to). You manually find the `spec.replicas` line and change the number:

```yaml
spec:
  replicas: 5   # change this value, then save and quit
```

Saving and exiting the editor applies the change immediately — Kubernetes reconciles the pod count right after you write the file.

## Quick comparison
![[Pasted image 20260813162226.png]]

If you just need to bump replica count, `kubectl scale` is the safer and faster choice. `kubectl edit` is more useful when you're already in there changing something else (labels, resource limits, image tag) and want to adjust replicas in the same pass.

### <mark style="background: #ABF7F7A6;">For describing/inspecting and deleting replicasets</mark>

## Describe a ReplicaSet

```bash
kubectl describe rs my-app-rs
```

or with the full kind name:

```bash
kubectl describe replicaset my-app-rs
```

If you don't know the exact name, list them first:

```bash
kubectl get rs
kubectl describe rs my-app-rs
```

This gives you a detailed human-readable dump — desired vs current vs ready replica counts, the pod template it uses, the label selector, and — most usefully for debugging — an **Events** section at the bottom showing recent scaling actions or failures (e.g. `FailedCreate` if pods can't be scheduled).

```
Name:         my-app-rs
Namespace:    default
Selector:     app=my-app
Labels:       app=my-app
Replicas:     3 current / 3 desired
Pods Status:  3 Running / 0 Waiting / 0 Succeeded / 0 Failed
Pod Template:
  Labels:  app=my-app
  Containers:
   app:
    Image:  my-app:1.0
    Port:   8080/TCP
Events:
  Type    Reason            Age   From                   Message
  ----    ------            ----  ----                   -------
  Normal  SuccessfulCreate  5m    replicaset-controller  Created pod: my-app-rs-x7f2k
```

## Delete a ReplicaSet

```bash
kubectl delete rs my-app-rs
```

By default this is a **cascading delete** — the ReplicaSet and all the pods it owns are removed.

If you want to delete only the ReplicaSet and leave its pods running (they become "orphaned," no longer managed by anything):

```bash
kubectl delete rs my-app-rs --cascade=orphan
```

Other ways to target it:

```bash
# delete by label instead of name
kubectl delete rs -l app=my-app

# delete straight from the YAML file that created it
kubectl delete -f replicaset.yaml
```

## Worth knowing

If the ReplicaSet was created by a Deployment (the common case), deleting the ReplicaSet directly is usually pointless — the Deployment controller will notice it's gone and immediately recreate it to match the desired state. To actually remove the pods in that scenario, you'd scale or delete the **Deployment**, not the ReplicaSet underneath it.