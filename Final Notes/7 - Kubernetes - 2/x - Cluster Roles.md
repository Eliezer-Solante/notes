![[Pasted image 20260821153536.png]]

![[Pasted image 20260821153613.png]]

![[Pasted image 20260821153655.png]]
![[Pasted image 20260821153727.png]]

to see the number of clusterrole or clusterrolebindings 
```bash
#Cluster Roles
kubectl get clusterrole --no-headers | wc -l
#Clust Role Bindings
kubectl get clusterrolebindings --no-headers | wc -l

```

---

# ClusterRole & ClusterRoleBinding

`ClusterRole` and `ClusterRoleBinding` extend RBAC beyond a single namespace — used for **cluster-scoped resources** or when permissions need to apply **across all namespaces**.

## Namespaced vs Cluster-Scoped Resources

Kubernetes resources fall into two categories:

|Namespaced|Cluster-Scoped|
|---|---|
|pods, jobs, services, roles|nodes, PV|
|replicasets, deployments, secrets|clusterroles 👑|
|rolebindings, configmaps, PVC|clusterrolebindings 👑|
||certificatesigningrequests|
||namespaces|

**Why this matters:** a plain `Role`/`RoleBinding` can only grant access to _namespaced_ resources within one namespace. Resources like `nodes` or `PV` don't belong to any namespace, so they can only be managed with `ClusterRole`/`ClusterRoleBinding`.

Check which category a resource falls into:

```bash
kubectl api-resources --namespaced=true    # pods, deployments, secrets, roles...
kubectl api-resources --namespaced=false   # nodes, PV, clusterroles, namespaces...
```

## 1. ClusterRole

Same rule syntax as `Role`, but not scoped to a namespace — `metadata.namespace` is omitted.

```yaml
# cluster-admin-role.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: cluster-administrator
rules:
- apiGroups: [""]
  resources: ["nodes"]
  verbs: ["list", "get", "create", "delete"]
```

```bash
kubectl create -f cluster-admin-role.yaml
```

This grants: view/create/delete **Nodes** — a cluster-scoped resource impossible to manage with a plain Role.

_(Another example from the notes — a "Storage Admin" ClusterRole would similarly grant view/create/delete on PVs and PVCs.)_

## 2. ClusterRoleBinding

Binds the `ClusterRole` to a user/group/service account **cluster-wide** — the permission applies across every namespace (and to cluster-scoped resources).

```yaml
# cluster-admin-role-binding.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: cluster-admin-role-binding
subjects:
- kind: User
  name: cluster-admin
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: cluster-administrator
  apiGroup: rbac.authorization.k8s.io
```

```bash
kubectl create -f cluster-admin-role-binding.yaml
```

**Result:** `dev-user` (bound as `cluster-admin`) → Cluster Admin role → can view/create/delete **Nodes**, cluster-wide.

## Command Equivalents

```bash
kubectl create clusterrole cluster-administrator \
  --verb=list,get,create,delete \
  --resource=nodes

kubectl create clusterrolebinding cluster-admin-role-binding \
  --clusterrole=cluster-administrator \
  --user=cluster-admin
```

## Viewing & Verifying

```bash
kubectl get clusterroles
kubectl get clusterrolebindings
kubectl describe clusterrole cluster-administrator
kubectl describe clusterrolebinding cluster-admin-role-binding

kubectl auth can-i delete nodes --as cluster-admin
# yes
```

## Recap: 4 Combinations

|Binding \ Role|Role (namespaced)|ClusterRole|
|---|---|---|
|**RoleBinding**|Permission limited to that Role's namespace|Grants ClusterRole's rules, but limited to the RoleBinding's namespace|
|**ClusterRoleBinding**|❌ Not allowed|Permission applies cluster-wide, across all namespaces|

This is why `ClusterRole` is often preferred even for reusable namespaced permissions (e.g. "pod-reader") — it can be bound narrowly with a `RoleBinding` per namespace, or broadly with a `ClusterRoleBinding`, without duplicating the rule definition.
