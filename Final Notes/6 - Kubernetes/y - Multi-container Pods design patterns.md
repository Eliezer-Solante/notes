
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
## Putting all three together
![[Pasted image 20260816233324.png]]

## How this maps back to sidecar/ambassador/adapter

The diagram you shared is about **structure and timing** — co-located vs sequential-and-done vs persistent-companion. Sidecar/ambassador/adapter (from the previous message) are about **role/purpose** — what job the helper container is doing.

So in practice:

- A **sidecar container** (structure) is almost always filling a **sidecar, ambassador, or adapter** role (purpose) — the terms overlap because "sidecar" is used both as the general structural pattern name _and_ as one of the three specific role patterns.
- A **regular init container** is structurally distinct — it's not really "sidecar/ambassador/adapter" in the role sense, since it doesn't persist; it's just prep work.
- **Co-located containers** is the umbrella category everything technically falls under, since sidecars are just co-located containers that happen to run for the full lifetime with a specific supporting role.

**Simplest way to hold it in your head:** Init containers = "run once, then get out of the way." Sidecars = "stick around and help." Co-located = the general term covering any multi-container pod, regardless of role.