![[Pasted image 20260821130040.png]]

![[Pasted image 20260821130131.png]]

![[Pasted image 20260821130140.png]]

![[Pasted image 20260821130155.png]]

![[Pasted image 20260821130207.png]]

![[Pasted image 20260821130339.png]]


![[Pasted image 20260821130415.png]]

![[Pasted image 20260821130529.png]]

![[Pasted image 20260821130732.png]]

![[Pasted image 20260821130753.png]]


# Kubernetes API Groups

The **kube-apiserver** exposes a REST API. All requests — from `kubectl`, dashboards, or `curl` — go through it.

## Top-level API Paths

```bash
curl https://kube-master:6443 -k
```

```json
{
  "paths": [
    "/api", "/api/v1",
    "/apis", "/apis/",
    "/healthz", "/logs",
    "/metrics", "/openapi/v2",
    "/swagger-2.0.0.json"
  ]
}
```

|Path|Purpose|
|---|---|
|`/metrics`, `/healthz`|Monitor cluster health|
|`/version`|View cluster version|
|`/api`|**Core** group|
|`/apis`|**Named** groups|
|`/logs`|Integrate with 3rd-party logging tools|

Check the cluster version:

```bash
curl https://kube-master:6443/version
```

## Core (`/api`) vs Named (`/apis`)

```
Core                    Named
/api                    /apis
```

### Core group → `/api/v1`

No group name in `apiVersion` — just `v1`. Contains the original, foundational resources:

```
/api/v1
 ├── namespaces   pods         rc(replicationcontrollers)
 ├── events       endpoints    nodes
 ├── bindings     PV           PVC
 └── configmaps   secrets      services
```

Query it directly:

```bash
curl https://kube-master:6443/api/v1/pods
```

```json
{
  "kind": "PodList",
  "apiVersion": "v1",
  "items": [
    {
      "metadata": { "name": "nginx-5c7588df-ghsbd", "namespace": "default" },
      "ownerReferences": [
        { "apiVersion": "apps/v1", "kind": "ReplicaSet", "name": "nginx-5c7588df" }
      ]
    }
  ]
}
```

Manifest:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  containers:
  - name: nginx
    image: nginx
```

### Named groups → `/apis/<group>/<version>`

```
/apis
 ├── /apps                    /v1  → deployments, replicasets, statefulsets
 ├── /extensions
 ├── /networking.k8s.io        /v1  → networkpolicies
 ├── /storage.k8s.io
 ├── /authentication.k8s.io
 └── /certificates.k8s.io      /v1  → certificatesigningrequests
```

Each resource under a group supports **verbs**: `list`, `get`, `create`, `delete`, `update`, `watch` — these map to RBAC rules.

Manifest:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deploy
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx
```

List all named groups:

```bash
curl https://localhost:6443/apis -k | grep "name"
```

```
"name": "extensions",
"name": "apps",
"name": "events.k8s.io",
"name": "authentication.k8s.io",
"name": "authorization.k8s.io",
"name": "autoscaling",
"name": "batch",
"name": "certificates.k8s.io",
"name": "networking.k8s.io",
"name": "policy",
"name": "rbac.authorization.k8s.io",
"name": "storage.k8s.io",
"name": "admissionregistration.k8s.io",
"name": "apiextensions.k8s.io",
"name": "scheduling.k8s.io",
"name": "coordination.k8s.io",
```

## Authentication Is Required

An unauthenticated request is rejected:

```bash
curl https://localhost:6443 -k
```

```json
{
  "kind": "Status",
  "status": "Failure",
  "message": "forbidden: User \"system:anonymous\" cannot get path \"/\"",
  "reason": "Forbidden",
  "code": 403
}
```

Pass client credentials to authenticate:

```bash
curl https://localhost:6443 -k \
  --key admin.key \
  --cert admin.crt \
  --cacert ca.crt
```

## kubectl proxy (auth made easy)

`kubectl proxy` starts a local proxy that already carries your kubeconfig credentials, so subsequent `curl` calls need no certs:

```bash
kubectl proxy
# Starting to serve on 127.0.0.1:8001
```

```bash
curl http://localhost:8001 -k
```

```json
{
  "paths": [
    "/api", "/api/v1", "/apis", "/apis/",
    "/healthz", "/logs", "/metrics",
    "/openapi/v2", "/swagger-2.0.0.json"
  ]
}
```

⚠️ **Don't confuse:**

```
kube-proxy   ≠   kubectl proxy
```

- **kube-proxy** — a node component that manages network rules for Service routing (iptables/IPVS).
- **kubectl proxy** — a local client-side tool that authenticates and forwards HTTP calls to the kube-apiserver.

## Quick Reference Commands

```bash
kubectl api-versions                    # all group/versions
kubectl api-resources                   # resources + kinds + group
kubectl api-resources --api-group=apps  # filter by group
kubectl explain deployment              # shows apiVersion/kind used
```