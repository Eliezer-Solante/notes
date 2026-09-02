## The core problem

Kubernetes RBAC has no native concept of AWS IAM, and AWS IAM has no native concept of Kubernetes RBAC. Something has to sit between them and translate:

> **AWS IAM identity → Kubernetes username + group**

That translator is the **AWS IAM Authenticator for Kubernetes**, running as part of the EKS control plane.

---

## Two separate access paths into the cluster

|Path|Flow|Purpose|
|---|---|---|
|**Human access**|Administrator → ALB → API Server → RBAC|A person running `kubectl`|
|**Workload/node access**|Node/Pod → AWS IAM → Authenticator → API Server|Nodes joining the cluster, IRSA-based Pod access|

Both paths converge at the same point: the IAM Authenticator, which must translate the AWS identity into something RBAC can evaluate — **before** Kubernetes RBAC ever decides what that identity is allowed to do.

```
Request → [AUTHENTICATION: IAM Authenticator] → [AUTHORIZATION: Kubernetes RBAC] → Action
```

---

## Evolution of the identity-mapping data source

### Stage 1 — `aws-auth` ConfigMap (legacy)

- A YAML ConfigMap stored in `kube-system` (the cluster's system namespace).
- The IAM Authenticator reads `mapRoles` / `mapUsers` entries from it to do the IAM → Kubernetes translation.
- **Drawback:** administrators had to manually edit this ConfigMap (directly, or via automation) to grant access.
    - No real API — just a raw file.
    - Prone to conflicts from concurrent edits.
    - Easy to make a breaking mistake and lock yourself out of the cluster.

### Stage 2 — Cluster Access APIs / Access Entries (modern)

- Introduces a proper, API-managed **"User Mappings"** layer — this is the **EKS Access Entries API**.
- Replaces the ConfigMap as the data source the Authenticator reads from.
- Managed via `aws eks create-access-entry`, `associate-access-policy`, etc. — no more direct YAML editing.
- Version-control friendly (Terraform/CloudFormation compatible).

### Stage 3 — Native IAM Integration (end state)

- The old `aws-auth` ConfigMap path is fully cut off (deprecated).
- The direct "IAM → ConfigMap" edit path is eliminated.
- All access flows cleanly through: **Administrator → API Server**, backed solely by **Access Entries (User Mappings)** feeding RBAC.

---

## Authentication modes (how the switch happens)

|Mode|Behavior|
|---|---|
|`CONFIG_MAP`|Legacy — only the `aws-auth` ConfigMap is used|
|`API_AND_CONFIG_MAP`|Transition mode — both sources checked, Access Entries take priority|
|`API`|Modern — only Access Entries are used, ConfigMap ignored entirely|

**Important:** this is a **one-way migration** — `CONFIG_MAP → API_AND_CONFIG_MAP → API` — you cannot switch back once you move forward.

---

## Where Kubernetes RBAC still fits in

Regardless of which identity-mapping method is used (ConfigMap or Access Entries), **Kubernetes RBAC is untouched** — it's the authorization layer that comes _after_ authentication:

- The Authenticator's job: confirm _who_ this is (AWS IAM identity → K8s username/group).
- RBAC's job: confirm _what_ that username/group is allowed to do (via Role/ClusterRole + RoleBinding/ClusterRoleBinding).

Optionally, with Access Entries you can skip managing RBAC objects entirely by using **EKS access policies** (predefined permission sets) instead.

---

## Summary — 4 key takeaways

1. This whole system exists to **manage cluster access for resource control**.
2. **Nodes and IAM-based workloads** need AWS API access to authenticate to the cluster, same as humans do.
3. **Authentication flows through a bridge component** (the IAM Authenticator) that links AWS IAM identities to Kubernetes roles.
4. **New Cluster Access APIs (Access Entries) replace the old `aws-auth` ConfigMap method** — the direction AWS is actively pushing all EKS clusters toward.

---

## Quick reference — old vs. new

||`aws-auth` ConfigMap|EKS Access Entries|
|---|---|---|
|Type|Raw YAML file in `kube-system`|AWS-managed API|
|Editing|Manual `kubectl edit`|`aws eks create-access-entry` / IaC|
|Conflict-safe|No — concurrent edits can clash|Yes|
|IaC-friendly|Awkward|Native (Terraform/CloudFormation)|
|AWS's recommended path|Being deprecated|**Recommended default going forward**|