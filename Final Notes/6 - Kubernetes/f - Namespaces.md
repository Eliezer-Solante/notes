![[Pasted image 20260813170738.png]]
Each namespace acts as its own scoped mini-cluster within the larger cluster — with its own workloads, its own quota cap, and its own network policy. The dotted line worth remembering: namespaces separate _names and scope_, but not _network traffic_, by default. That isolation only exists once a NetworkPolicy is explicitly applied.


<mark style="background: #ABF7F7A6;">To get the pods on each namespace</mark>
![[Pasted image 20260813171100.png]]

<mark style="background: #ABF7F7A6;">Creating pod in default or a specific namespace</mark>
![[Pasted image 20260813171334.png]]
![[Pasted image 20260813171341.png]]

<mark style="background: #BBFABBA6;">To create a pod in the default namespace</mark>
![[Pasted image 20260813171321.png]]

<mark style="background: #BBFABBA6;">To create a pod in a specific namespace (example: `dev` namespace)</mark>
![[Pasted image 20260813171458.png]]

if you want to create a pod to a namespace all the time place the `namespace` to the YAML file 
![[Pasted image 20260813171652.png]]


#### <mark style="background: #ABF7F7A6;">Creating a Namespace</mark>
<mark style="background: #BBFABBA6;">YAML file</mark>
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: my-app-namespace
  labels:
    environment: production
```
Notice there's no `spec` field of real substance here — much like ConfigMap/Secret, a Namespace is just declared into existence via `metadata`.
![[Pasted image 20260813171819.png]]
<mark style="background: #BBFABBA6;">you can also create a namespace directly using create </mark>
![[Pasted image 20260813171911.png]]

#### <mark style="background: #ABF7F7A6;">Switching to another namespace</mark>
![[Pasted image 20260813172246.png]]![[Pasted image 20260813172217.png]]

To deploy something _into_ a namespace, you reference it in that object's own metadata:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
  namespace: my-app-namespace
spec:
  replicas: 3
  ...
```

## <mark style="background: #FF5582A6;">Common commands</mark>

```bash
# create a namespace imperatively
kubectl create namespace my-app-namespace

# or apply from YAML
kubectl apply -f namespace.yaml

# list all namespaces
kubectl get namespaces
kubectl get ns

# describe one (shows resource quotas, limit ranges if set)
kubectl describe ns my-app-namespace

# list resources within a specific namespace
kubectl get pods -n my-app-namespace
kubectl get all -n my-app-namespace

# switch your default working namespace (avoids typing -n every time)
kubectl config set-context --current --namespace=my-app-namespace

# delete a namespace — this cascades and deletes EVERYTHING inside it
kubectl delete ns my-app-namespace

# list all pods in all namespaces existing
kubectl get pods --all-namespaces
#or
kubectl get pods -A

# to list services in a namespace
kubectl get svc -n=marketing

```

## A few things worth knowing

- **Default namespaces on every cluster**: `default` (where things land if you don't specify one), `kube-system` (cluster-internal components like the DNS server and scheduler), `kube-public`, and `kube-node-lease`.
- **Deleting a namespace is destructive and cascading** — every Pod, Deployment, Service, ConfigMap, etc. inside it gets deleted too. There's no confirmation prompt, so this is a common accidental-deletion mistake in production.
- **DNS and cross-namespace communication**: Services get a DNS name like `my-service.my-app-namespace.svc.cluster.local` — pods in a different namespace need the full name; pods in the _same_ namespace can just use `my-service`.
- **Namespaces don't provide network isolation by default** — pods in different namespaces can still talk to each other over the network unless you add a `NetworkPolicy` to restrict it.


## ResourceQuota — capping what a namespace can consume

Without a quota, one team's namespace could consume all the cluster's CPU/memory and starve everyone else.

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: team-a-quota
  namespace: team-a
spec:
  hard:
    requests.cpu: "4"
    requests.memory: 8Gi
    limits.cpu: "8"
    limits.memory: 16Gi
    pods: "20"
```

```bash
kubectl apply -f quota.yaml
kubectl describe quota team-a-quota -n team-a   # shows used vs hard limit
```
![[Pasted image 20260813172355.png]]
If a new pod would push the namespace over any of these limits, it's rejected at creation time.

## LimitRange — setting defaults within a namespace

Without this, a pod with no `resources` field has no cap at all. A LimitRange sets defaults so people don't forget:

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: default-limits
  namespace: team-a
spec:
  limits:
    - default:
        cpu: "500m"
        memory: "256Mi"
      defaultRequest:
        cpu: "250m"
        memory: "128Mi"
      type: Container
```

## NetworkPolicy — the isolation namespaces don't give you by default

This is important since it directly contradicts a common assumption: **pods in different namespaces can talk to each other freely unless you explicitly block it.**

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-from-other-namespaces
  namespace: team-a
spec:
  podSelector: {}          # applies to all pods in this namespace
  policyTypes:
    - Ingress
  ingress:
    - from:
        - podSelector: {}   # only allow traffic from pods within team-a
```

```bash
kubectl apply -f netpol.yaml
kubectl get networkpolicy -n team-a
```

## Putting it together — a realistic namespace setup

```bash
kubectl create namespace team-a
kubectl apply -f quota.yaml -n team-a
kubectl apply -f limits.yaml -n team-a
kubectl apply -f netpol.yaml -n team-a
kubectl apply -f deployment.yaml -n team-a
```

## Quick recap table

|Object|Purpose|
|---|---|
|Namespace|Logical grouping/scoping of resources|
|ResourceQuota|Caps total CPU/memory/object count for the namespace|
|LimitRange|Sets default and max limits per individual pod/container|
|NetworkPolicy|Restricts network traffic in/out of the namespace|
|RBAC (Role + RoleBinding)|Restricts _who_ can do what within the namespace|

---


![[Pasted image 20260813170649.png]]
This image is showing how Kubernetes DNS resolves service names differently depending on which namespace you're calling _from_.

## What's in the picture

Two namespaces are shown as separate "houses" — `default` and `dev` — each containing its own `web-pod`, `db-service`, and (in `default`) a `web-deployment`. Notice both namespaces have a service named `db-service` — that's allowed, because namespaces give you separate naming scopes, exactly like we discussed earlier.

The two terminal snippets at the bottom show `mysql.connect()` calls to that same-named service, but with different connection strings.

## Why the connection strings differ

**Left terminal — `mysql.connect("db-service")`** This is called from _inside_ the `default` namespace. Kubernetes DNS lets you use the short name when you're calling a service **within the same namespace** — the cluster's DNS resolver automatically assumes "search here first" and resolves it to the local `db-service`.

**Right terminal — `mysql.connect("db-service.dev.svc.cluster.local")`** This is the **fully qualified domain name (FQDN)**. If `web-pod` in `default` needed to reach the `db-service` living in `dev`, the short name `db-service` alone would resolve to the _local_ one (in `default`) — the wrong one. So you have to spell out the full path to cross namespace boundaries.

## The FQDN format, broken down
![[Pasted image 20260813170923.png]]

```
db-service   .dev          .svc          .cluster.local
   ↑             ↑             ↑                ↑
service name   namespace   resource type   cluster domain
```

- **`db-service`** — the Service name
- **`dev`** — the namespace it lives in
- **`svc`** — indicates this is a Service (as opposed to a pod)
- **`cluster.local`** — the default base domain for the cluster's internal DNS

## The core lesson this diagram is making

This ties directly into something from our namespace discussion: **namespaces scope naming, not just isolation**. Two services can share the exact same name (`db-service`) in different namespaces without conflict, precisely because DNS resolution is namespace-aware. Short names only work for same-namespace lookups; anything crossing a namespace boundary needs the FQDN.



A **Namespace** is a way to divide a single cluster into multiple virtual clusters — it's a scoping mechanism for organizing and isolating resources (Pods, Deployments, Services, etc.), not a physical separation like a VM.

## Why they exist

- **Multi-team / multi-project isolation** — e.g. `team-a`, `team-b` can each deploy resources with the same names without colliding.
- **Environment separation** — `dev`, `staging`, `production` often live as separate namespaces in the same cluster.
- **Resource quotas** — you can cap CPU/memory/object counts per namespace.
- **RBAC scoping** — permissions can be granted per-namespace rather than cluster-wide.

Not everything is namespaced, though — Nodes, PersistentVolumes, and Namespaces themselves are **cluster-scoped**, meaning they exist outside any namespace.