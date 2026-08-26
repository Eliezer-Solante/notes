![[Pasted image 20260826092441.png]]

![[Pasted image 20260826092526.png]] ![[Pasted image 20260826092612.png]]


It's a managed service from Amazon Web Services that lets you run Kubernetes — the open-source container orchestration system originally developed by Google — without having to install, operate, and maintain your own Kubernetes control plane.

**Key points about EKS:**

- **Managed control plane**: AWS runs and manages the Kubernetes control plane components (API server, etcd, scheduler, controller manager) across multiple Availability Zones for high availability. You don't have to patch, upgrade, or scale these yourself.
- **You still manage worker nodes** (mostly): You typically manage the EC2 instances (or use Fargate for serverless compute) that run your actual application containers, though EKS offers managed node groups to simplify this too.
- **Standard Kubernetes**: EKS runs upstream, certified Kubernetes, so tools, plugins, and workflows that work with Kubernetes elsewhere (kubectl, Helm, etc.) generally work with EKS too — it's not a proprietary fork.
- **Integrates with AWS services**: EKS ties into IAM (for authentication/authorization), VPC (networking), ELB (load balancing), EBS/EFS (storage), CloudWatch (monitoring), and more.
- **Compute options**: You can run workloads on EC2 instances you manage, EKS Managed Node Groups (AWS handles provisioning/lifecycle), or AWS Fargate (fully serverless, no node management at all).

In short, EKS takes the operational burden of running Kubernetes' control plane off your plate while giving you a standard, portable Kubernetes environment tightly integrated with the AWS ecosystem.