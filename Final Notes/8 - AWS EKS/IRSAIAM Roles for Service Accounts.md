> ## Documentation Index
> Fetch the complete documentation index at: https://notes.kodekloud.com/llms.txt
> Use this file to discover all available pages before exploring further.

# IRSAIAM Roles for Service Accounts

> This article explains how to configure IAM Roles for Service Accounts (IRSA) in Amazon EKS for secure AWS API access.

In Amazon EKS, pods typically rely on node IAM permissions or sidecar proxies to access AWS APIs. IRSA (IAM Roles for Service Accounts) streamlines this by binding an IAM role directly to a Kubernetes ServiceAccount. This approach enforces least-privilege access at the pod level and removes the need to distribute AWS credentials manually.

In this guide, you’ll learn how to configure IRSA on EKS for a pod that reads objects from an S3 bucket.

***

## Overview of the IRSA Authentication Flow

1. EKS provisions an OIDC identity provider for the cluster.
2. You create an IAM role with a trust policy for that OIDC provider.
3. EKS injects a mutating webhook into the control plane.
4. You annotate a Kubernetes ServiceAccount with the IAM role’s ARN.
5. When your pod starts, the webhook mounts a projected token and sets AWS environment variables.
6. The AWS SDK inside the pod:
   * Retrieves a JWT from the OIDC provider.
   * Calls STS `AssumeRoleWithWebIdentity` using that JWT.
   * Receives temporary AWS credentials.
7. Your application uses those credentials to call S3.

***

## 1. Configure the OIDC Provider

When you create an EKS cluster, AWS automatically registers an OIDC identity provider. Retrieve its issuer URL:

```bash theme={null}
aws eks describe-cluster \
  --name my-cluster \
  --query "cluster.identity.oidc.issuer" \
  --output text
```

Use this URL in your IAM role trust policy (see next step).

***

## 2. Create an IAM Role with Trust Policy

Define a trust policy that allows your EKS OIDC provider to assume the role:

```json theme={null}
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::123456789012:oidc-provider/oidc.eks.region.amazonaws.com/id/EXAMPLED539D4633E53DE1B71EXAMPLE"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "oidc.eks.region.amazonaws.com/id/EXAMPLED539D4633E53DE1B71EXAMPLE:sub": "system:serviceaccount:app:s3-reader"
        }
      }
    }
  ]
}
```

<Callout icon="lightbulb" color="#1CB2FE">
  Replace the OIDC issuer URL, AWS account ID, and Kubernetes namespace/serviceaccount in the trust policy to match your environment.
</Callout>

Attach an S3 read permissions policy:

```json theme={null}
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["s3:GetObject", "s3:ListBucket"],
      "Resource": ["arn:aws:s3:::my-bucket", "arn:aws:s3:::my-bucket/*"]
    }
  ]
}
```

***

## 3. EKS Mutating Webhook Injection

EKS deploys a mutating webhook that intercepts pod creation. It mounts a projected service account token and injects AWS environment variables (`AWS_ROLE_ARN`, `AWS_WEB_IDENTITY_TOKEN_FILE`, etc.) into pods that reference an annotated ServiceAccount.

<Frame>
  ![The image illustrates a setup for webhook and service account annotation, showing the integration of AWS IAM, an EKS cluster, and AWS S3 using IRSA.](https://kodekloud.com/kk-media/image/upload/v1752862889/notes-assets/images/AWS-EKS-IRSAIAM-Roles-for-Service-Accounts/webhook-service-account-aws-eks-s3.jpg)
</Frame>

***

## 4. Annotate a Kubernetes ServiceAccount

Create a ServiceAccount annotated with your IAM role ARN:

```yaml theme={null}
apiVersion: v1
kind: ServiceAccount
metadata:
  name: s3-reader
  namespace: app
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::123456789012:role/S3ReadRole
```

Reference it in a Deployment:

```yaml theme={null}
apiVersion: apps/v1
kind: Deployment
metadata:
  name: reader
  namespace: app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: reader
  template:
    metadata:
      labels:
        app: reader
    spec:
      serviceAccountName: s3-reader
      containers:
      - name: app
        image: amazonlinux
        command:
          - sh
          - -c
          - aws s3 ls s3://my-bucket
```

***

## 5. Runtime Token Exchange

1. Pod mounts the projected token and sets AWS variables.
2. AWS SDK fetches a JWT from the EKS OIDC provider.
3. It calls STS `AssumeRoleWithWebIdentity` with the token.
4. STS validates the token against the IAM role’s trust policy.
5. STS returns temporary AWS credentials.
6. The application uses these credentials to access S3.

***

## 6. Scalability and Operational Considerations

| Concern             | Description                                                                   | Recommendation                                              |
| ------------------- | ----------------------------------------------------------------------------- | ----------------------------------------------------------- |
| OIDC provider limit | AWS accounts support up to 100 OIDC identity providers.                       | Consolidate clusters or share providers if possible.        |
| Trust relationships | Roles cannot trust an OIDC provider before the cluster exists.                | Provision clusters before creating IAM roles.               |
| Multi-cluster setup | Each EKS cluster has its own issuer ARN.                                      | Add multiple issuer ARNs to a single IAM role trust policy. |
| STS rate limits     | Calls to `AssumeRoleWithWebIdentity` have per-role/issuer throttling.         | Create additional IAM roles or rotate usage patterns.       |
| Granularity         | Fine-grained roles per namespace/ServiceAccount increase management overhead. | Use naming conventions and automation to manage many roles. |

<Callout icon="triangle-alert" color="#FF6B6B">
  If you run more than 100 EKS clusters in one AWS account, you will hit the OIDC provider limit. Consider using fewer providers or multiple AWS accounts.
</Callout>

<Frame>
  ![The image is a diagram illustrating scalability concerns and limitations in an AWS EKS environment, showing interactions between components like EKS Clusters, IAM, OIDC, and AWS S3. It highlights the use of IRSA, webhooks, and service accounts within the architecture.](https://kodekloud.com/kk-media/image/upload/v1752862890/notes-assets/images/AWS-EKS-IRSAIAM-Roles-for-Service-Accounts/aws-eks-scalability-diagram-components.jpg)
</Frame>

<Frame>
  ![The image is a diagram illustrating management and maintenance challenges with AWS EKS clusters across different regions, showing connections between AWS IAM, ARN, and STS.](https://kodekloud.com/kk-media/image/upload/v1752862891/notes-assets/images/AWS-EKS-IRSAIAM-Roles-for-Service-Accounts/aws-eks-management-challenges-diagram.jpg)
</Frame>

***

## 7. When to Use IRSA

IRSA is ideal when you need per-pod, least-privilege AWS access without distributing node IAM keys. However, at scale or in multi-cluster deployments, managing OIDC providers and IAM roles can become complex.

* **Use IRSA when**
  * You require tight access control per ServiceAccount.
  * You want to eliminate node-wide IAM credentials.
* **Consider alternatives when**
  * Your cluster fleet exceeds OIDC limits.
  * You prefer a simpler EC2 metadata or sidecar proxy solution.

<Callout icon="lightbulb" color="#1CB2FE">
  AWS EKS Pod Identity provides another AWS-native approach that may scale better in large, multi-region environments.
</Callout>

***

## Links and References

* [Amazon EKS Documentation](https://docs.aws.amazon.com/eks/latest/userguide/what-is-eks.html)
* [Kubernetes ServiceAccount](https://kubernetes.io/docs/tasks/configure-pod-container/configure-service-account/)
* [AWS STS AssumeRoleWithWebIdentity API](https://docs.aws.amazon.com/STS/latest/APIReference/API_AssumeRoleWithWebIdentity.html)
* [Setting Up IAM Roles for Service Accounts (IRSA)](https://docs.aws.amazon.com/eks/latest/userguide/iam-roles-for-service-accounts.html)

<CardGroup>
  <Card title="Watch Video" icon="video" cta="Learn more" href="https://learn.kodekloud.com/user/courses/aws-eks/module/1bbca08a-6f11-4fbe-89db-aeee53fccb0d/lesson/bf6e4d70-5762-47ba-bbc3-673fc821b0bc" />
</CardGroup>
