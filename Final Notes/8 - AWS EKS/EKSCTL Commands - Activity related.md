Here's the updated version with custom-named IAM roles included as a pre-step, since — as noted — eksctl doesn't let you set custom role names purely through the YAML config.

## Process Overview

1. Pre-create IAM roles (if custom names are required)
2. Write a `ClusterConfig` YAML defining VPC, subnets, IAM/OIDC, addons, and node group
3. Run one `eksctl` command to provision everything
4. Confirm kubeconfig and verify nodes/pods
5. Deploy workload

---

## 1. IAM Roles (pre-created, with custom names)

**Cluster IAM Role:**
```bash
cat > cluster-trust.json <<'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": { "Service": "eks.amazonaws.com" },
      "Action": "sts:AssumeRole"
    }
  ]
}
EOF

aws iam create-role \
  --role-name my-eks-cluster-role \
  --assume-role-policy-document file://cluster-trust.json

aws iam attach-role-policy \
  --role-name my-eks-cluster-role \
  --policy-arn arn:aws:iam::aws:policy/AmazonEKSClusterPolicy
```

**Node IAM Role:**
```bash
cat > node-trust.json <<'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": { "Service": "ec2.amazonaws.com" },
      "Action": "sts:AssumeRole"
    }
  ]
}
EOF

aws iam create-role \
  --role-name my-eks-node-role \
  --assume-role-policy-document file://node-trust.json

aws iam attach-role-policy --role-name my-eks-node-role --policy-arn arn:aws:iam::aws:policy/AmazonEKSWorkerNodePolicy
aws iam attach-role-policy --role-name my-eks-node-role --policy-arn arn:aws:iam::aws:policy/AmazonEKS_CNI_Policy
aws iam attach-role-policy --role-name my-eks-node-role --policy-arn arn:aws:iam::aws:policy/AmazonEC2ContainerRegistryReadOnly

# Grab the ARN for use in the eksctl config below
NODE_ROLE_ARN=$(aws iam get-role --role-name my-eks-node-role --query Role.Arn --output text)
echo "$NODE_ROLE_ARN"
```

> **Note:** eksctl has no config field to make the *cluster* use a pre-existing custom-named role — it always generates its own (`eksctl-<cluster>-cluster-ServiceRole-xxxx`). The block above is for reference/consistency with manual IAM setups, but only the **node role** can actually be injected into the eksctl-created node group (via `iam.instanceRoleARN`, shown below).

---

## 2. The Manifest (`cluster-config.yaml`)

```yaml
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig

metadata:
  name: my-cluster
  region: us-east-1
  version: "1.36"
  tags:
    env: training
    team: my-team

# --- Networking (VPC, subnets, IGW, NAT, route tables — all automatic) ---
vpc:
  cidr: "10.10.0.0/24"
  nat:
    gateway: Single
  subnets:
    public:
      us-east-1a: { cidr: "10.10.0.0/28" }
      us-east-1b: { cidr: "10.10.0.16/28" }
    private:
      us-east-1a: { cidr: "10.10.0.64/26" }
      us-east-1b: { cidr: "10.10.0.128/26" }

# --- OIDC provider (needed for IRSA) ---
iam:
  withOIDC: true

# --- Addons ---
addons:
  - name: vpc-cni
    version: latest
  - name: coredns
    version: latest
  - name: kube-proxy
    version: latest
  - name: aws-ebs-csi-driver
    version: latest
    wellKnownPolicies:
      ebsCSIController: true   # eksctl auto-creates a scoped IRSA role for this addon

# --- Managed node group, using the pre-created custom-named node role ---
managedNodeGroups:
  - name: my-ng-01
    instanceType: t3.medium
    amiFamily: AmazonLinux2023
    desiredCapacity: 2
    minSize: 0
    maxSize: 3
    privateNetworking: true
    iam:
      instanceRoleARN: "arn:aws:iam::<ACCOUNT_ID>:role/my-eks-node-role"
    tags:
      env: training
```

---

## 3. Commands

```bash
# Create everything: VPC, subnets, IGW, NAT, route tables, OIDC provider,
# addons, and node group (using the pre-created node role) — in one shot
eksctl create cluster -f cluster-config.yaml

# Confirm/refresh kubeconfig (eksctl does this automatically, but explicit check is good practice)
aws eks update-kubeconfig --region us-east-1 --name my-cluster

# Verify cluster and nodes
kubectl get nodes -o wide
kubectl get pods -A

# Verify cluster/addon status via AWS CLI
aws eks describe-cluster --name my-cluster --region us-east-1 \
  --query 'cluster.{Name:name,Status:status,Version:version}' --output table
aws eks list-addons --cluster-name my-cluster --region us-east-1 --output table

# Deploy a test workload
kubectl create deployment nginx-test --image=nginx:latest --replicas=2
kubectl rollout status deployment/nginx-test
```

---

## 4. Cleanup command (for reference)

```bash
eksctl delete cluster -f cluster-config.yaml
```
This tears down the cluster, node group, and OIDC-created IRSA roles via CloudFormation. **It does not delete the manually pre-created `my-eks-cluster-role` / `my-eks-node-role`** — those must be deleted separately with `aws iam delete-role` (after detaching policies) if you're fully cleaning up.

---

## Key notes to remember

- **VPC + subnets + IGW + NAT + route tables** — created automatically from the `vpc:` block, no separate `aws ec2` commands needed.
- **Cluster IAM role** — eksctl always generates its own; a pre-created custom-named role cannot be injected for the cluster itself.
- **Node IAM role** — *can* be a pre-created custom-named role, referenced via `managedNodeGroups[].iam.instanceRoleARN`.
- **OIDC provider + IRSA roles** — handled via `iam.withOIDC: true` plus `wellKnownPolicies` for common addons; no manual trust-policy JSON needed for those.
- **Addon sequencing** (addons `DEGRADED` before nodes exist) is handled internally by eksctl.