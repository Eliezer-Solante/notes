> ## Documentation Index
> Fetch the complete documentation index at: https://notes.kodekloud.com/llms.txt
> Use this file to discover all available pages before exploring further.

# Pod Identity

> This article explains Pod Identity on Amazon EKS, a method for assigning AWS IAM roles to Kubernetes pods, improving permissions management and integration.

Pod Identity—often called IRSA v2—is the latest, native EKS method for assigning AWS IAM roles to Kubernetes pods. In this guide, you’ll learn how Pod Identity improves on Kube2IAM and IRSA by streamlining permissions management, removing annotation overhead, and integrating role mappings directly into the EKS control plane.

***

## Recap: Kube2IAM vs. IRSA

Before diving into Pod Identity, here’s a quick comparison of earlier approaches:

| Approach | Mechanism                                                      | Drawbacks                                             |
| -------- | -------------------------------------------------------------- | ----------------------------------------------------- |
| Kube2IAM | iptables intercept to metadata endpoint + proxy                | Complex setup, relies on node IAM role                |
| IRSA     | OIDC provider + mutating webhook + service account annotations | Requires OIDC provider management and pod annotations |

Kube2IAM uses iptables rules to redirect 169.254.169.254 traffic to a proxy, which then fetches credentials based on pod annotations. IRSA introduced an OIDC identity provider and webhook that injects AWS\_WEB\_IDENTITY\_TOKEN\_FILE and AWS\_ROLE\_ARN into pod environments—trading complexity of iptables for OIDC setup.

***

## Introducing Pod Identity

Pod Identity modernizes IRSA by embedding role-to-service-account mappings in the EKS control plane. Key benefits include:

* No external OIDC provider
* No pod-level AWS annotations
* Local credential proxy via DaemonSet
* Support for tag-based access control (ABAC)

### Architecture Overview

Pods continue calling the AWS SDK normally. A mutating webhook in the EKS control plane injects environment variables pointing to a local, node-hosted proxy. A privileged DaemonSet uses host networking to listen for SDK requests, then interacts with the EKS API to issue temporary credentials.

<Frame>
  ![The image illustrates the concept of "Pod Identity" with a focus on creating proxied credentials and using a privileged pod with host networking, represented by gears and a daemon set icon.](https://kodekloud.com/kk-media/image/upload/v1752862892/notes-assets/images/AWS-EKS-Pod-Identity/pod-identity-privileged-pod-credentials.jpg)
</Frame>

This proxy authenticates using the node’s EC2 instance role and retrieves the correct IAM role for each service account.

<Frame>
  ![The image illustrates the concept of "Pod Identity" with elements like Daemon Set, Service Accounts, and AWS IAM, highlighting the need for credential association between service accounts and IAM.](https://kodekloud.com/kk-media/image/upload/v1752862893/notes-assets/images/AWS-EKS-Pod-Identity/pod-identity-daemon-set-aws-iam.jpg)
</Frame>

***

## Role-to-Service-Account Association

With Pod Identity, you no longer annotate ServiceAccount objects. Instead, run an EKS CLI command:

```bash theme={null}
aws eks associate-service-account-role \
  --cluster-name my-cluster \
  --namespace default \
  --service-account my-app-sa \
  --role-arn arn:aws:iam::123456789012:role/MyPodRole
```

EKS stores this mapping internally. Your pod specs stay clean—no AWS-specific annotations needed.

***

## Tag-Based Credentials (ABAC)

Pod Identity fully supports AWS attribute-based access control. Tag your AWS resources (for example, `Environment=dev` on an S3 bucket) and reference those tags in your IAM policies. At credential time, EKS evaluates both:

1. The service account ➔ IAM role mapping
2. Any resource tags defined in IAM policies

<Frame>
  ![The image illustrates a concept of "Tag-Based Credentials" in AWS, showing a user, an S3 bucket, and a message about writing to the bucket if it has the right tags.](https://kodekloud.com/kk-media/image/upload/v1752862894/notes-assets/images/AWS-EKS-Pod-Identity/tag-based-credentials-aws-s3-diagram.jpg)
</Frame>

***

## Request Flow

1. Submit a Pod spec referencing a ServiceAccount (no AWS annotations).
2. EKS webhook mutates the Pod spec, injecting `AWS_ROLE_ARN` and `AWS_WEB_IDENTITY_TOKEN_FILE` toward a local IP (e.g., `169.254.x.x`).
3. At runtime, the AWS SDK in the pod calls the local proxy.
4. The proxy reads the service account token, contacts the EKS control plane, and requests credentials.
5. EKS checks the service-account→role mapping and any tag-based policies, then issues temporary credentials.
6. The proxy returns credentials to the SDK, which uses them for AWS API calls (e.g., `PutObject` to S3).

<Frame>
  ![The image illustrates the workflow of an EKS cluster, showing components like the control plane, service accounts, IAM roles, and a daemon set interacting with a workload and an S3 bucket.](https://kodekloud.com/kk-media/image/upload/v1752862896/notes-assets/images/AWS-EKS-Pod-Identity/eks-cluster-workflow-control-plane.jpg)
</Frame>

***

## Limitations & Coexistence

<Callout icon="triangle-alert" color="#FF6B6B">
  Pod Identity is exclusive to EKS. If you operate Kubernetes on EC2 without the EKS control plane, continue using Kube2IAM or similar solutions.
</Callout>

You can run both IRSA and Pod Identity in the same cluster. EKS will prioritize Pod Identity for credential injection but still support IRSA workloads during your migration.

<Frame>
  ![The image is a comparison chart showing "Kube2iam," "IRSA," and "Pod Identity," with checkmarks next to "IRSA" and "Pod Identity," and an icon labeled "Webhook" below "Pod Identity."](https://kodekloud.com/kk-media/image/upload/v1752862897/notes-assets/images/AWS-EKS-Pod-Identity/kube2iam-irsa-pod-identity-chart.jpg)
</Frame>

### Feature Comparison

| Feature                    | Kube2IAM | IRSA | Pod Identity |
| -------------------------- | :------: | :--: | :----------: |
| Native EKS Integration     |     –    |   ✔  |       ✔      |
| ServiceAccount Annotations |     ✔    |   ✔  |       –      |
| OIDC Provider Required     |     –    |   ✔  |       –      |
| Local Credential Proxy     |     –    |   –  |       ✔      |
| Tag-Based Policies         |     –    |   –  |       ✔      |

***

## Next Steps

Now that Pod Identity handles IAM at the pod level, explore how to control **ingress** and **egress** with VPC security groups for Kubernetes workloads.

## Links and References

* [Amazon EKS Pod Identity Guide](https://docs.aws.amazon.com/eks/latest/userguide/pod-identity.html)
* [Kubernetes Service Accounts](https://kubernetes.io/docs/tasks/configure-pod-container/configure-service-account/)
* [AWS IAM ABAC](https://docs.aws.amazon.com/IAM/latest/UserGuide/access_tags.html)

<CardGroup>
  <Card title="Watch Video" icon="video" cta="Learn more" href="https://learn.kodekloud.com/user/courses/aws-eks/module/1bbca08a-6f11-4fbe-89db-aeee53fccb0d/lesson/024e43e2-a20b-41d6-a070-56873b9b0be2" />
</CardGroup>
