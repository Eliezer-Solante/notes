![[Pasted image 20260821131749.png]]

![[Pasted image 20260821131756.png]]

![[Pasted image 20260821131801.png]]

![[Pasted image 20260821131808.png]]

**![[Pasted image 20260821131817.png]]**

![[Pasted image 20260821131915.png]]
![[Pasted image 20260821142957.png]]

To inspect what authorization modes configures on the cluster:
`kubectl describe pod kube-apiserver-controlplane -n kube-system`


![[Pasted image 20260821143008.png]]

![[Pasted image 20260821143015.png]]
# Kubernetes Authorization

Authentication proves **who you are**; authorization decides **what you're allowed to do**.

## Why Authorization?

Different identities need different permissions on the same cluster:

```
Admins  → kubectl get pods    ✅         kubectl delete node worker-2   ✅ Deleted
Devs    → kubectl get pods    ✅         kubectl delete node worker-2   ❌ Forbidden
Bots    → kubectl get pods    ❌ Forbidden: User "Bot-1" cannot list "pods"
```

Without authorization, any authenticated user/bot could do anything — including deleting nodes.

## Authorization Mechanisms

```
Node → ABAC → RBAC → Webhook
```

### 1. Node Authorizer

Special-purpose authorizer that only applies to **kubelets**. Grants each kubelet (identified by cert group `SYSTEM:NODES`, user `system:node:<name>`) permission to:

|Read|Write|
|---|---|
|Services|Node status|
|Endpoints|Pod status|
|Nodes|Events|
|Pods||

### 2. ABAC (Attribute-Based Access Control)

Permissions defined as static JSON policy files, one rule per line, tied directly to a user/group + resource:

```json
{"kind": "Policy", "spec": {"user": "dev-user", "namespace": "*", "resource": "pods", "apiGroup": "*"}}
{"kind": "Policy", "spec": {"group": "dev-users", "namespace": "*", "resource": "pods", "apiGroup": "*"}}
{"kind": "Policy", "spec": {"user": "security-1", "namespace": "*", "resource": "csr", "apiGroup": "*"}}
```

**Downside:** hard to manage — requires editing files + restarting the API server for every change.

### 3. RBAC (Role-Based Access Control) — most common

Permissions attached to **roles**, and roles attached to users/groups — not directly to individuals. Much easier to maintain.

```
dev-user, dev-user-2, dev-users (group)  →  Role: Developer  →  view/create/delete Pods
security-1                                →  Role: Security   →  view/create CSR
```

**Manifest — Role (namespace-scoped):**

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: default
  name: developer
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list", "create", "delete"]
```

**Manifest — RoleBinding:**

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: developer-binding
  namespace: default
subjects:
- kind: Group
  name: dev-users
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: developer
  apiGroup: rbac.authorization.k8s.io
```

Commands:

```bash
kubectl create role developer --verb=get,list,create,delete --resource=pods
kubectl create rolebinding dev-binding --role=developer --group=dev-users
kubectl auth can-i delete pods --as=dev-user
```

### 4. Webhook

Delegates the authorization decision to an **external service** (e.g., Open Policy Agent):

```
User → kube-apiserver → Webhook (OPA)
"User dev-user requested read access to Pods. Should I allow?" → "I checked. Yes!"
```

## Configuring Authorization Mode

Set on the kube-apiserver via `--authorization-mode`. Modes are evaluated **in the order listed**, left to right:

```bash
ExecStart=/usr/local/bin/kube-apiserver \
  --authorization-mode=Node,RBAC,Webhook \
  ...
```

Available modes:

```
AlwaysAllow | Node | ABAC | RBAC | Webhook | AlwaysDeny
```

- `AlwaysAllow` — default if unset; allows all requests (insecure, avoid in production).
- `AlwaysDeny` — blocks everything.

### Chained evaluation
#### The Three Verdicts

Each authorizer in the chain returns one of:

| Verdict        | What happens                                                                  |
| -------------- | ----------------------------------------------------------------------------- |
| **Allow**      | Request is immediately granted. Chain stops. No further modules checked.      |
| **Deny**       | Request is immediately rejected. Chain stops. No further modules checked.     |
| **No Opinion** | Authorizer abstains — passes the request to the **next** module in the chain. |

So it's not that a "deny" moves to the next module — a **denial is final**, same as an allow. It's the **"no opinion"** case that continues the chain.

#### Why This Matters for `Node,RBAC,Webhook`

```
--authorization-mode=Node,RBAC,Webhook
```

```
User → Node authorizer
         ├── Request is about kubelet-related resources → Allow/Deny (final)
         └── Not a kubelet request → No Opinion → passes to RBAC

       → RBAC authorizer
         ├── Matching Role/RoleBinding found → Allow (final)
         ├── Explicit deny rule matched → Deny (final)
         └── No matching rule at all → No Opinion → passes to Webhook

       → Webhook authorizer
         └── External service (e.g., OPA) makes final call
```

If **every** module returns "no opinion," the request is denied by default (Kubernetes fails closed).

#### Matching the diagram

```
User → Node (no opinion) → RBAC (decides) → back to User
```

That's exactly why Webhook isn't reached in Image 9 — RBAC already returned a verdict (allow or deny), so the chain stopped before Webhook was ever consulted.

#### Quick way to test this yourself

bash

```bash
kubectl auth can-i delete pods --as=dev-user
kubectl auth can-i delete pods --as=system:node:worker-1
```

Both hit the same chain, but resolve at different stages depending on who's asking and what resource is targeted.