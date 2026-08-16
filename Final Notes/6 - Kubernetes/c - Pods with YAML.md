Kubernetes uses YAML as its primary way to describe **desired state** — you write what you want to exist, and the cluster works to make reality match it (this is the "declarative" approach, as opposed to issuing step-by-step commands).

## The basic structure

Every Kubernetes YAML file (called a "manifest") shares four top-level fields:

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

- **apiVersion** — which version of the Kubernetes API this object belongs to (e.g. `v1` for core objects like Pods, `apps/v1` for Deployments)
- **kind** — the type of object: `Pod`, `Deployment`, `Service`, `ConfigMap`, etc.
- **metadata** — identifying info: name, namespace, labels, annotations
- **spec** — the actual desired state, and its shape depends entirely on `kind`

## Why a Deployment YAML nests a pod template

Notice the `template` block above — that's a full pod spec embedded inside the Deployment. This is the pattern you were asking about earlier: the Deployment doesn't run containers directly, it manages a set of identical pods (the `replicas: 3` line), and each pod is stamped out from that template. This is how horizontal scaling actually works in YAML terms.

## A few other things worth knowing

- **`kubectl apply -f file.yaml`** is how you hand a manifest to the cluster — it diffs your file against current state and reconciles the difference.
- **Multiple objects in one file**: separate them with `---`. Common to define a Deployment and its Service together.
- **Indentation is meaningful** (it's YAML) — a misplaced two spaces will silently change which field something belongs to, so this is the most common source of manifest bugs.
- **`kubectl get pod my-pod -o yaml`** dumps the live YAML of anything running in the cluster, useful for seeing what Kubernetes filled in automatically (status fields, defaults you didn't specify).


Here's a standalone Pod manifest — the raw `kind: Pod` you'd rarely use directly in production (Deployments manage pods for you), but it's the clearest way to see what a pod actually contains.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-app-pod
  labels:
    app: my-app
spec:
  containers:
    - name: app
      image: my-app:1.0
      ports:
        - containerPort: 8080
      resources:
        requests:
          cpu: "250m"
          memory: "128Mi"
        limits:
          cpu: "500m"
          memory: "256Mi"
      env:
        - name: ENVIRONMENT
          value: "production"
    - name: log-sidecar
      image: fluent-bit:2.0
      volumeMounts:
        - name: shared-logs
          mountPath: /var/log/app
  volumes:
    - name: shared-logs
      emptyDir: {}
```

## Walking through the fields

- **`spec.containers`** — a list, matching what we discussed earlier: here `app` is the main container and `log-sidecar` is the helper. They run side by side inside the same pod, sharing network and storage.
- **`image`** — which container image to pull and run, usually `name:tag` from a registry like Docker Hub or a private one.
- **`resources.requests` / `limits`** — requests are what the scheduler guarantees when placing the pod on a node; limits are the hard ceiling the container can't exceed. `250m` means 0.25 CPU cores (m = millicores). Getting these wrong is a very common source of pods getting evicted or throttled.
- **`env`** — environment variables injected into that specific container.
- **`volumes`** (pod-level) + **`volumeMounts`** (container-level) — this is exactly the "shared storage" from earlier. `volumes` declares storage at the pod level; each container decides where to mount it. Here, `emptyDir` is a temp directory that lives as long as the pod does — both containers can read/write to `/var/log/app` and see each other's files.

## Why you rarely write `kind: Pod` yourself

Bare pods aren't self-healing — if the node dies, the pod is just gone. That's why in practice you almost always wrap this same `spec` inside a Deployment's `template` block (like last message), or a `StatefulSet`/`DaemonSet` for other patterns, so a controller keeps recreating pods to match your desired count.


---

| ==**kind**==            | ==**apiVersion**==   | ==**metadata (typical use)**== | ==**spec (typical contents)**==                           |
| ----------------------- | -------------------- | ------------------------------ | --------------------------------------------------------- |
| Pod                     | v1                   | name, labels                   | containers, volumes, resources                            |
| Deployment              | apps/v1              | name, labels                   | replicas, selector, template (pod spec)                   |
| StatefulSet             | apps/v1              | name, labels                   | replicas, selector, template, volumeClaimTemplates        |
| DaemonSet               | apps/v1              | name, labels                   | selector, template (runs one pod per node)                |
| ReplicaSet              | apps/v1              | name, labels                   | replicas, selector, template                              |
| Service                 | v1                   | name, labels                   | selector, ports, type (ClusterIP/NodePort/LoadBalancer)   |
| ConfigMap               | v1                   | name, namespace                | data (key-value config, no spec field)                    |
| Secret                  | v1                   | name, namespace                | data / stringData (base64-encoded values, no spec field)  |
| Ingress                 | networking.k8s.io/v1 | name, annotations              | rules, tls, backend routing                               |
| Namespace               | v1                   | name                           | (no spec field needed)                                    |
| Job                     | batch/v1             | name, labels                   | template, completions, backoffLimit                       |
| CronJob                 | batch/v1             | name, labels                   | schedule, jobTemplate                                     |
| PersistentVolumeClaim   | v1                   | name, namespace                | accessModes, resources.requests.storage, storageClassName |
| HorizontalPodAutoscaler | autoscaling/v2       | name                           | scaleTargetRef, minReplicas, maxReplicas, metrics         |

A couple of notes on the table: `ConfigMap` and `Secret` don't use `spec` at all — they use `data` directly, since they're just storing values rather than describing running behavior. And `apiVersion` isn't arbitrary per kind — each `kind` is registered under a specific API group, so mismatching them (e.g. writing `kind: Deployment` under `apiVersion: v1`) is a common error `kubectl apply` will reject.

Run after configuring the YAML file
![[Pasted image 20260813130938.png]]

Run to check the pod
![[Pasted image 20260813131046.png]]

![[Pasted image 20260813134304.png]]

if you edited the YAML file, and updating the pod without delete and creating a new pod
![[Pasted image 20260813134445.png]]
or

#### A Note on Editing Existing Pods

In any of the practical quizzes, if you are asked to **edit an existing POD**, please note the following:

- If you are given a pod definition file, edit that file and use it to create a new pod.
- **If you are not given a pod definition file**, you may extract the definition to a file using the below command:

```
kubectl get pod  -o yaml > pod-definition.yaml
```

- Then edit the file to make the necessary changes, delete, and re-create the pod.
- To modify the properties of the pod, you can utilize the

```
kubectl edit pod 
```

command. Please note that only the properties listed below are editable.

- spec.containers[*].image
- spec.initContainers[*].image
- spec.activeDeadlineSeconds
- spec.tolerations
- spec.terminationGracePeriodSeconds