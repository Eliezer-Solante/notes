
![[Pasted image 20260821141241.png]]

![[Pasted image 20260821141228.png]]
![[Pasted image 20260821141300.png]]

![[Pasted image 20260821141506.png]]

![[Pasted image 20260821141459.png]]

To check the roles that exist in all namespaces:
`kubectl get roles -A`

---

# Role-Based Access Control (RBAC) — Full Walkthrough

## 1. Define a Role

```yaml
# developer-role.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: developer
  namespace: default
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["list", "get", "create", "update", "delete"]
- apiGroups: [""]
  resources: ["ConfigMap"]
  verbs: ["create"]
```

```bash
kubectl create -f developer-role.yaml
```

This gives the `developer` role permission to view/create/update/delete **Pods** and create **ConfigMaps**, scoped to the `default` namespace.

## 2. Bind the Role to a User

```yaml
# devuser-developer-binding.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: devuser-developer-binding
subjects:
- kind: User
  name: dev-user
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: developer
  apiGroup: rbac.authorization.k8s.io
```

```bash
kubectl create -f devuser-developer-binding.yaml
```

**Result:** `dev-user` → bound to `Developer` role → can view/create/delete PODs, create ConfigMaps.

## 3. Viewing RBAC Objects

```bash
$ kubectl get roles
NAME        AGE
developer   4s

$ kubectl get rolebindings
NAME                        AGE
devuser-developer-binding   24s
```

Inspect the Role's rules:

```bash
$ kubectl describe role developer
Name:         developer
Labels:       <none>
Annotations:  <none>
PolicyRule:
  Resources   Non-Resource URLs   Resource Names   Verbs
  ---------   -----------------   --------------   -----
  ConfigMap   []                  []                [create]
  pods        []                  []                [get watch list create delete]
```

Inspect the RoleBinding:

```bash
$ kubectl describe rolebinding devuser-developer-binding
Name:         devuser-developer-binding
Labels:       <none>
Annotations:  <none>
Role:
  Kind:  Role
  Name:  developer
Subjects:
  Kind   Name       Namespace
  ----   ----       ---------
  User   dev-user
```

## 4. Checking Access with `kubectl auth can-i`

```bash
# As the currently authenticated user (e.g. cluster-admin)
$ kubectl auth can-i create deployments
yes
$ kubectl auth can-i delete nodes
no

# Impersonating another user with --as
$ kubectl auth can-i create deployments --as dev-user
no                     # role has no rule for deployments

$ kubectl auth can-i create pods --as dev-user
yes                    # matches the pods rule

$ kubectl auth can-i create pods --as dev-user --namespace test
no                     # Role/RoleBinding only apply in "default" namespace
```

**Key takeaway:** a `Role`/`RoleBinding` grants access **only within the namespace it's created in**. To grant access to `dev-user` in `test`, you'd need another RoleBinding (or Role) in that namespace — or use a `ClusterRole` + `RoleBinding` combo.

## 5. Restricting to Specific Resources — `resourceNames`

Use `resourceNames` to limit a rule to specific named objects instead of _all_ resources of that type.

```yaml
# developer-role.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: developer
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "create", "delete"]
  resourceNames: ["blue", "orange"]
```

Given pods named `blue`, `orange`, `green`, `purple`, `pink` — the `developer` role can only `get`/`create`/`delete` the pods **named `blue` or `orange`**; the rest (`green`, `purple`, `pink`) are inaccessible even though the resource type (`pods`) matches.

```bash
kubectl auth can-i delete pods/blue --as dev-user     # yes
kubectl auth can-i delete pods/green --as dev-user    # no
```

## Quick Reference

| Task            | Command                                                             |
| --------------- | ------------------------------------------------------------------- |
| Create role     | `kubectl create role <name> --verb=<v1,v2> --resource=<r1,r2>`      |
| Create binding  | `kubectl create rolebinding <name> --role=<role> --user=<user>`     |
| List roles      | `kubectl get roles`                                                 |
| List bindings   | `kubectl get rolebindings`                                          |
| Inspect role    | `kubectl describe role <name>`                                      |
| Inspect binding | `kubectl describe rolebinding <name>`                               |
| Test permission | `kubectl auth can-i <verb> <resource> --as <user> --namespace <ns>` |

