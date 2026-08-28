# Lab Summary: EKS Cluster Access via IAM User Authentication

## Goal

To demonstrate how to grant an **external AWS IAM user** access to a Kubernetes (EKS) cluster by mapping that IAM identity into Kubernetes RBAC — going from "no access" to scoped, read-only access on pods and deployments.

## Method Used

**Cluster Access (aws-auth ConfigMap + Kubernetes RBAC)**

This is the classic **`aws-auth` ConfigMap** approach (also called EKS Access Entries in newer setups), _not_ IRSA, Pod Identity, or Security Groups. Key distinction:

- **IRSA / Pod Identity** → grant AWS permissions _to a pod/service account_ (workload → AWS).
- **Security Groups for Pods** → control _network_ traffic to/from pods.
- **Cluster Access (this lab)** → grant an **IAM principal (user/role) access to the Kubernetes API** itself, then use native K8s RBAC (Role/RoleBinding) to scope what that principal can do inside the cluster.

---

## Steps & Commands

### 1. Verify current user's cluster access

```bash
kubectl config view
kubectl get pods --all-namespaces
```

Confirms the current kubeconfig identity (likely cluster admin/creator) can already reach the cluster.

### 2. Create an IAM user + access key

```bash
aws iam create-user --user-name iamuser-eksuser
aws iam create-access-key --user-name iamuser-eksuser | tee /tmp/create_output
```

Creates a new IAM identity outside the cluster, with credentials saved for later use (e.g., configuring a separate AWS CLI profile to test as this user).

### 3. Map the IAM user into Kubernetes via `aws-auth-cm.yaml`

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: aws-auth
  namespace: kube-system
data:
  mapRoles: |
    - rolearn: arn:aws:iam::891377054545:role/eks-demo-node
      username: system:node:{{EC2PrivateDNSName}}
      groups:
        - system:bootstrappers
        - system:nodes
  mapUsers: |
    - userarn: arn:aws:iam::891377054545:user/iamuser-eksuser
      username: iamuser-eksuser
```

```bash
kubectl apply -f aws-auth-cm.yaml
```

This tells EKS: "authenticate this IAM user, and inside Kubernetes, call them `iamuser-eksuser`." **Authentication only** — no permissions (authorization) granted yet.

### 4. Verify access (should be denied)

```bash
kubectl auth can-i get pod --as iamuser-eksuser
```

**Expected: `no`** — user is authenticated but has no RBAC permissions yet.

### 5. Grant permissions via Role + RoleBinding

**Role** (`user-role.yaml`):

```yaml
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: iamuser-eks-role
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["list", "get", "watch"]
- apiGroups: ["extensions", "apps"]
  resources: ["deployments"]
  verbs: ["get", "list", "watch"]
```

```bash
kubectl create -f user-role.yaml
```

**RoleBinding** (`user-role-binding.yaml`):

```yaml
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: iamuser-eks-binding
subjects:
- kind: User
  name: iamuser-eksuser
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: iamuser-eks-role
  apiGroup: rbac.authorization.k8s.io
```

```bash
kubectl create -f user-role-binding.yaml
```

### 6. Verify access again (should now be allowed)

```bash
kubectl auth can-i get pod --as iamuser-eksuser
```

**Expected: `yes`**

---

## Key Takeaway

Access to EKS = **two layers**:

1. **Authentication** (AWS IAM → `aws-auth` ConfigMap) — "who are you?"
2. **Authorization** (Kubernetes RBAC → Role/RoleBinding) — "what can you do?"

Being mapped in `aws-auth` alone is not enough; RBAC rules must also be explicitly granted (least-privilege by default).