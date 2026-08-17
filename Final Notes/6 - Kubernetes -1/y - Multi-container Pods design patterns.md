
![[Pasted image 20260816232928.png]]
![[Pasted image 20260816232915.png]]

Good — this image is actually mapping out the **three ways multiple containers relate to each other inside a pod**, which is a slightly different (and more foundational) axis than sidecar/ambassador/adapter. Those three patterns describe _what a helper container's job is_; the diagram you shared describes _how containers are structured/scheduled relative to each other_. Let's break down each one.

## 1. Co-located Containers

This is the general, catch-all case — just **multiple containers running in the same pod**, sharing network and (optionally) storage, with no special ordering or role implied.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: co-located-demo
spec:
  containers:
  - name: container-a
    image: app-a:latest
  - name: container-b
    image: app-b:latest
```
![[Pasted image 20260816233156.png]]
- Both start at roughly the same time.
- No dependency relationship enforced — Kubernetes doesn't guarantee A starts before B.
- This is the **base mechanism** — sidecar/ambassador/adapter are all just _specific use cases_ of co-located containers, distinguished by role, not by any different YAML structure.

**When to use:** any two containers that just need to run together and share network/storage, with no strict startup order requirement.

## 2. Regular Init Containers

Containers that run **before** the main container(s), sequentially, and **must complete successfully (exit 0)** before the next one — or the main container — starts. The green checkmark in the diagram represents this "must finish first" gate.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: init-demo
spec:
  initContainers:
  - name: wait-for-db
    image: busybox
    command: ['sh', '-c', 'until nc -z db-service 5432; do sleep 2; done']
  - name: run-migrations
    image: migrate-tool:latest
    command: ['./migrate.sh']
  containers:
  - name: app
    image: myapp:latest
```
![[Pasted image 20260816233142.png]]
- Init containers run **one at a time, in order** (not in parallel).
- Each must exit successfully before the next starts.
- Once all init containers finish, they **stop running entirely** — they don't persist alongside the main container.
- If an init container fails, the kubelet retries it according to the pod's `restartPolicy`.

**When to use:** one-time setup tasks — waiting for a dependency, running migrations, cloning a repo into a shared volume, generating config files before the app starts.

## 3. Sidecar Containers

A helper container that runs **alongside** the main container for the pod's full lifetime — this is the pattern we discussed in depth last message (log shippers, service mesh proxies, etc.).

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: sidecar-demo
spec:
  containers:
  - name: app
    image: myapp:latest
  - name: log-shipper
    image: fluentd:latest
```

**Modern native version (K8s 1.28+)** — declared as an init container but with `restartPolicy: Always`, so it starts early (in init order) but then keeps running for the pod's entire life, unlike a regular init container which exits and disappears:

```yaml
spec:
  initContainers:
  - name: log-shipper
    image: fluentd:latest
    restartPolicy: Always   # ← this is what makes it a persistent sidecar, not a one-shot init container
  containers:
  - name: app
    image: myapp:latest
```
![[Pasted image 20260816233305.png]]

Deployment sample
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  labels:
    app: myapp
spec:
  replicas: 1
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
        - name: myapp
          image: alpine:latest
          command: ['sh', '-c', 'while true; do echo "logging" >> /opt/logs.txt; sleep 1; done']
          volumeMounts:
            - name: data
              mountPath: /opt
      initContainers:
        - name: logshipper
          image: alpine:latest
          # Setting restartPolicy: Always makes this a sidecar container.
          restartPolicy: Always
          command: ['sh', '-c', 'tail -F /opt/logs.txt']
          volumeMounts:
            - name: data
              mountPath: /opt
      volumes:
        - name: data
          emptyDir: {}
```
## Putting all three together
![[Pasted image 20260816233324.png]]

## How this maps back to sidecar/ambassador/adapter

The diagram you shared is about **structure and timing** — co-located vs sequential-and-done vs persistent-companion. Sidecar/ambassador/adapter (from the previous message) are about **role/purpose** — what job the helper container is doing.

So in practice:

- A **sidecar container** (structure) is almost always filling a **sidecar, ambassador, or adapter** role (purpose) — the terms overlap because "sidecar" is used both as the general structural pattern name _and_ as one of the three specific role patterns.
- A **regular init container** is structurally distinct — it's not really "sidecar/ambassador/adapter" in the role sense, since it doesn't persist; it's just prep work.
- **Co-located containers** is the umbrella category everything technically falls under, since sidecars are just co-located containers that happen to run for the full lifetime with a specific supporting role.

**Simplest way to hold it in your head:** Init containers = "run once, then get out of the way." Sidecars = "stick around and help." Co-located = the general term covering any multi-container pod, regardless of role.


---

## SAMPLE SITUATIONS

# K8s Task Notes — Native Sidecar for Log Shipping (Filebeat + Elasticsearch)

## Scenario

- Namespace: `elastic-stack`
- Existing `app` pod writes logs to `/log/app.log` (via `log-volume`, hostPath).
- Goal: ship logs to Elasticsearch using a sidecar container.

## Requirements

1. Add sidecar container named `sidecar` to the `app` pod.
2. Image: `kodekloud/filebeat-configured`.
3. Mount existing `log-volume` at `/var/log/event-simulator/` in the sidecar.
4. Define sidecar as a **native sidecar** — i.e. under `initContainers`, with `restartPolicy: Always`.
5. Don't modify the existing `app` container or volume config.
6. Pod must be deleted + recreated to apply changes (containers aren't hot-patchable).

## Key concept: native sidecars

- Since K8s 1.28+ (stable 1.29), sidecars are defined under `initContainers` but with `restartPolicy: Always` set on that specific container.
- This makes it start before the main container, then **keep running continuously** alongside it (unlike a normal init container, which runs once and exits).

## Solution

```yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    name: app
  name: app
  namespace: elastic-stack
spec:
  initContainers:
  - name: sidecar
    image: kodekloud/filebeat-configured
    restartPolicy: Always
    volumeMounts:
      - name: log-volume
        mountPath: /var/log/event-simulator
  containers:
  - image: kodekloud/event-simulator
    name: app
    resources: {}
    volumeMounts:
    - mountPath: /log
      name: log-volume
  volumes:
  - hostPath:
      path: /var/log/webapp
      type: DirectoryOrCreate
    name: log-volume
```

## Steps to apply

```bash
kubectl get pod app -n elastic-stack -o yaml > app.yaml   # back up / edit
kubectl delete pod app -n elastic-stack
kubectl apply -f app.yaml
```

## Gotcha

- Sidecar goes in `initContainers`, **not** `containers` — the `restartPolicy: Always` on that container is what distinguishes it from a regular one-shot init container.

---
Additional Notes

# Understanding Init and Sidecar Containers in Kubernetes

In a **multi-container Pod**, each container is expected to run a process that stays alive for the **entire lifecycle of the Pod**.

For example, in a Pod with a **web application** and a **logging agent**, both containers are expected to remain active throughout the Pod’s lifecycle. The process in the logging agent container must stay alive as long as the web application is running. If any main container fails and the Pod's `restartPolicy` is `Always` or `OnFailure`, the **entire Pod is restarted**.

---

## Pod Restart Behavior in Multi-Container Pods

It's important to understand how restarts work in **multi-container Pods**:

- If any **main container** (i.e., containers listed under `spec.containers`) exits and the Pod's `restartPolicy` is set to `Always` or `OnFailure`, **all containers in the Pod are restarted**.
- Kubernetes does not restart individual containers within a Pod. Instead, it treats the Pod as a single unit of execution and **restarts the entire Pod** if needed.

This applies only to main containers, not init containers. Init containers are always run to completion **before** main containers begin and are not restarted individually.

However, sometimes you may want to run a process that completes and exits before the main containers start. This is where **init containers** are used.

Examples include:

- A script that pulls code or binaries from a repository before the application starts.
- A script that waits for an external service (like a database) to become available.

---

## What is an Init Container?

An **init container** is a special container that runs **before the main containers** in a Pod. Each init container must **succeed (exit 0)** before the next one is started. Once all init containers complete, the regular containers start **simultaneously**.

They are configured similarly to other containers but are placed in the `initContainers` section of the Pod spec.

> If any init container fails, the entire Pod is restarted and **all init containers are rerun from the beginning**.

---

## ✅ Using Init Containers

```
apiVersion: v1
kind: Pod
metadata:
  name: myapp-pod
  labels:
    app: myapp
spec:
  initContainers:
    - name: init-myservice
      image: busybox:1.31
      command: ["sh", "-c", "until nslookup myservice; do echo waiting for myservice; sleep 2; done;"]
    - name: init-mydb
      image: busybox:1.31
      command: ["sh", "-c", "until nslookup mydb; do echo waiting for mydb; sleep 2; done;"]
  containers:
    - name: myapp-container
      image: busybox:1.28
      command: ["sh", "-c", "echo The app is running! && sleep 3600"]
```

---

## Native Sidecar Containers (Kubernetes 1.33+)

Starting with Kubernetes v1.33, **sidecar containers** are natively supported. This allows sidecar containers to follow a defined lifecycle relative to the main containers in the Pod — without requiring entrypoint hacks.

### ✳️ How Native Sidecars Work

- Declared using the `restartPolicy: Always` field inside the `initContainers` block.
- Kubernetes treats such containers as **sidecars**, ensuring they:
    - Start **before** main containers.
    - Run **alongside** them.
    - **Shut down after** the main containers complete.

---

### ✅ Example: Native Sidecar Configuration

```
apiVersion: v1
kind: Pod
metadata:
  name: sidecar-example
spec:
  initContainers:
    - name: sidecar-logger
      image: busybox:1.31
      restartPolicy: Always
      command: ["sh", "-c", "while true; do echo Sidecar running; sleep 10; done"]
  containers:
    - name: main-app
      image: busybox:1.31
      command: ["sh", "-c", "echo Main app starting; sleep 60"]
```

In this setup:

- The **`sidecar-logger`** container behaves like a sidecar, though declared in `initContainers`.
- It uses `restartPolicy: Always` to stay alive throughout the Pod lifecycle.
- Kubernetes starts the sidecar first, waits for readiness, then starts the main app.