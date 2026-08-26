# EKS Notes

A study guide to Amazon Elastic Kubernetes Service (EKS), organized from fundamentals through networking, storage, secrets, load balancing, compute, resiliency, and cluster maintenance.

---

## 1. EKS Fundamentals

### What is EKS

EKS (Elastic Kubernetes Service) is AWS's **hosted/managed service for Kubernetes**. Every Kubernetes cluster is made of two halves:

- **Control plane** — etcd, the API server, the scheduler, the controller manager. The "brain" of the cluster.
- **Data plane** — the worker nodes that actually run your pods.

With EKS, AWS draws a clean line down the middle of that split:

![[Pasted image 20260826143521.png]]

The two sides are bridged by cross-account **Elastic Network Interfaces (ENIs)** — think of it as plugging a cable between two separate networks so they can talk to each other.

**Why use EKS instead of running Kubernetes yourself on EC2?** You could run your own control plane on EC2 (tools like `kops` or `kubespray` have done this for years). You'd get full control — your own scheduler flags, your own upgrade schedule, any cluster size you want. The catch: you now own _everything_, including the scariest part — **etcd**, the distributed database behind every Kubernetes cluster. If it's lost, your cluster is effectively gone. Many teams switch to a hosted service the first time they have to restore an etcd backup themselves. EKS hands that responsibility — and the operational burden of keeping the control plane highly available — to AWS.

> 💡 Most companies aren't running AWS _just_ for Kubernetes — they're already using RDS, S3, load balancers, Route 53, etc. EKS is the easiest way to wire a Kubernetes cluster into the rest of that AWS ecosystem.

### Common Use Cases

- **Microservices architectures** — breaking a monolith into independently deployable services with built-in service discovery, load balancing, and rolling updates.
- **CI/CD & DevOps pipelines** — containerized build/test/deploy pipelines (Jenkins, GitLab CI, ArgoCD, Tekton), pairing well with GitOps workflows.
- **Batch processing & data pipelines** — ETL and scheduled jobs via Kubernetes Jobs/CronJobs, scaling workers up during processing and down afterward.
- **Machine learning / AI workloads** — training and serving models, especially with GPU-backed node groups (e.g. via Kubeflow).
- **Hybrid & multi-cloud deployments** — because EKS runs standard upstream Kubernetes, workloads can be designed to be portable (EKS Anywhere supports on-prem clusters with the same tooling).
- **High-availability, auto-scaling web apps** — e-commerce, SaaS, etc., relying on Horizontal Pod Autoscaler / Karpenter for scaling and self-healing.
- **Event-driven workloads** — combined with Fargate, run spiky/unpredictable traffic without managing idle servers.
- **Multi-tenant SaaS platforms** — namespaces, resource quotas, and network policies isolate customers/teams within a single cluster.
- **Legacy application modernization** — "lift and shift" containerization onto EKS.
- **Edge & IoT workloads** — via AWS Outposts or EKS Anywhere, running Kubernetes closer to edge locations.

### Architecture

EKS is a **regional service**, but a region is really a group of Availability Zones (separate data centers). Control plane pieces are spread across a **minimum of three AZs** so an outage in one AZ doesn't take the cluster down — etcd needs quorum, so it always keeps at least three copies alive and runs leader election.

Beyond the standard Kubernetes control plane pieces, an EKS cluster also sets up a few EKS-specific extras:

- **OIDC endpoint** — authentication for anything that needs to call into AWS from inside the cluster.
- **CloudWatch logging** — opt-in, but the plumbing is already there.
- **aws-auth / EKS Access Entries API** — maps AWS IAM identities to Kubernetes RBAC.
- **Node Groups & Add-ons APIs** — technically part of the EKS control-plane API, but they create resources (nodes, workloads) inside _your_ account.

### Deployment Options

There's no single right way to create a cluster — it comes down to what your team already knows:

- **Console** — works, but is consistently the most-complained-about experience, since EKS touches so many other AWS services (IAM, VPC, load balancers) that you end up bouncing between tabs.
- **CloudFormation** — hand-written YAML, or generated via the CDK. Good if you're an AWS-native shop already.
- **Terraform** — probably the most popular option. AWS maintains a set of community modules called **EKS Blueprints**. You own your own state file and access since it's not a managed service.
- **Everything else** — Pulumi, Cluster API, and a dozen other tools that ultimately script the same CloudFormation/Terraform/CLI calls for you.

> 💡 Whatever tool you pick, you'll need fairly broad IAM permissions to create a cluster — EKS touches EC2, VPCs, security groups, load balancers, and IAM roles all at once.

### Tools needed for EKS

- **kubectl** — the standard Kubernetes CLI.
- **aws-iam-authenticator** — a required `kubectl` plugin. The EKS API server sits behind a load balancer that requires AWS-signed credentials, so this plugin turns your IAM identity into something `kubectl` can use to authenticate.
- **eksctl / Terraform / CloudFormation / CDK** — whichever infrastructure tool you choose for creating and managing the cluster (see Deployment Options above).
- **AWS CLI** — for scripting direct API/CLI calls when you want more granular control than a wrapper tool provides.

### Networking (overview)

Every node and pod pulls its IP address from your VPC's subnets, using Elastic Network Interfaces (ENIs) to plug into the network. Because subnets are AZ-bound, so are the nodes and pods that live in them. _(Full details in Section 2 — EKS Networking.)_

### Authentication (overview)

AWS IAM identities and Kubernetes RBAC are two separate systems — something has to map one to the other. This can happen through the legacy **aws-auth ConfigMap** or the modern **EKS Access Entries API**. _(Full details in Section 7 — Cluster Access.)_

---

## 2. EKS Networking

![[Pasted image 20260826143809.png]]
### How networking works

Every node and pod gets its IP address from the subnets in your VPC — and because subnets are bound to a single Availability Zone, so is every node and pod inside them. Nodes reach the network through **Elastic Network Interfaces (ENIs)**, which behave just like a physical network card plugged into a server.

**The ENI / IP-address bottleneck:** EC2 instance types have hard limits on how many ENIs they can attach and how many IP addresses each ENI can hold (e.g. a small instance might support only 4–5 ENIs at 4–5 IPs each). Once you've scheduled enough pods to exhaust that pool, the VPC CNI has to register and attach an entirely new ENI — which is **not instant** (10–15 seconds or more), especially if the subnet is running low on free addresses and has to wait for IP leases to expire.

Two settings help the CNI stay ahead of that churn:

- **`WARM_ENI_TARGET`** — keep N extra ENIs pre-attached and ready before you need them.
- **`WARM_IP_TARGET`** — keep N extra IP addresses ready regardless of how many ENIs that takes (handles mixed instance sizes more gracefully).

### Prefix Delegation

The fix AWS recommends for (almost) everyone. Instead of handing out one IP address per ENI slot, prefix delegation assigns a whole **/28 block (16 addresses)** as a single route.

- One ENI slot now unlocks **16 pod IPs** instead of one.
- A small instance that used to max out at ~4 usable pod IPs can jump to ~64.
- Enable it with the **`ENABLE_PREFIX_DELEGATION`** environment variable on the VPC CNI.

### IPv6

If you can start a brand-new cluster on IPv6 (dual-stack VPC), running out of IP addresses stops being a problem entirely.

- Each node gets an **/80 block** = **100 trillion addresses** (more than the _entire_ IPv4 internet, which only has ~4 billion addresses total).
- Pods only get an **IPv6** address in EKS — there's no dual-stack at the pod level.
- A local **169.254.x.x** address on the node handles NAT translation for the rare case where a pod still needs to call a legacy IPv4 endpoint.
- **Watch out for:** many pods on one node all sharing that single outbound IPv4 NAT path — this can become a bottleneck if your surrounding infrastructure is still largely IPv4.

### Network Policies

The standard Kubernetes way to control which pods can talk to which other pods (both ingress and egress).

- The AWS VPC CNI implements network policies using **eBPF** — a small program running in the node's Linux kernel, not a sidecar. This makes it a genuinely application-native firewall.
- **Benefit:** application teams can manage their own traffic rules as part of the same manifests they deploy, instead of filing a ticket with a separate network/firewall team every time an endpoint changes.
- Different CNI providers (e.g. Calico) implement network policy differently — behavior may differ if you're not using the default VPC CNI.

---

## 3. EKS Storage

Kubernetes constructs like `emptyDir` are fine for scratch space, but the data disappears the moment the pod does. For anything durable (like a StatefulSet), you need storage that survives outside the pod's lifecycle — which in AWS usually means EBS or EFS, wired in through a **CSI driver** (Container Storage Interface).

Every CSI driver ships two pieces:

- A **controller Deployment** that manages the full lifecycle of volumes against the AWS API.
- A **DaemonSet** that runs on every node to actually perform the mount.

### EKS EBS (Elastic Block Store)

Block storage — like plugging in a new hard drive.

- **Zone-bound**: if your autoscaler creates a replacement node in the wrong AZ, the pod will sit unschedulable while the autoscaler cycles through AZs trying to find the right one.
- 1 pod ↔ 1 volume (single-writer).
- Very low latency (same-AZ).
- Multiple volume types (GP2/GP3/IO) with different throughput/cost trade-offs; supports snapshots.
- Great fit for databases and other single-writer workloads.

### EKS EFS (Elastic File System)

File storage (NFS) — "just give me files," no formatting or disk management needed.

- **Regional**, not zonal — many pods across many AZs can mount the same volume simultaneously.
- Classic NFS pitfalls apply: file locking can hang concurrent writers if you have high-concurrency write patterns.
- Needs IAM + security group setup before it can be mounted.
- **Dynamic provisioning is not available on Fargate or Windows nodes.**
- Great fit for shared config, shared uploads, or workloads that need multi-AZ read/write access to the same files.

### EKS Other Storage

- **FSx for Lustre** — AWS-managed Lustre, for high-throughput HPC-style workloads.
- **S3** — many apps talk to S3 natively via SDK. There's also a newer **S3 Mount Point CSI driver** that presents a bucket as a filesystem — handy, but **not POSIX-compliant** (renames, `mkdir`, etc. behave differently than a real filesystem).
- **Local volumes (NVMe)** — the fastest storage available in AWS, but fully ephemeral; data disappears when the node does unless separately snapshotted.
- **In-cluster storage** — Longhorn, Rook/Ceph, OpenEBS, etc. Runs entirely inside the cluster using node-local storage; more portable across clouds/on-prem, but more operational burden (you own the replication, backups, and failure handling).

---

## 4. EKS Secrets

### EKS Secrets Intro

A **Kubernetes Secret** looks like a ConfigMap, except the value is **base64-encoded** before being sent to the API server.

> ⚠️ **Important:** base64 is not encryption. It's trivially reversible — it obscures a value from a casual glance, but does **not** protect it. Kubernetes Secrets are stored as base64 text, so anyone with API/etcd access (or the right RBAC) can decode them instantly.

For anything genuinely sensitive (API keys, root passwords), don't rely on native Secrets alone — layer strong RBAC and namespace separation on top, or better, move the secret outside the cluster entirely.

### Kubernetes Secrets Options

Recommended pattern for real secrets:

1. **Store secrets outside the cluster:**
    - **AWS Secrets Manager** — the natural choice inside AWS.
    - **HashiCorp Vault** — common if you're multi-cloud or on-prem.
2. **Mount them into pods using the Secrets Store CSI Driver** (a generic CSI driver for secret stores, with backends for most major cloud providers plus Vault).
    - Can mount secrets as files on disk, or
    - Sync them as short-lived, temporary Kubernetes Secrets (created and deleted automatically alongside the pod's lifecycle) when a workload needs environment variables instead of a volume.
3. **Benefits over native Secrets:**
    - Real encryption at rest, managed outside Kubernetes.
    - Automatic **secret rotation** — the workload always reads the latest value without needing AWS SDK credentials baked into the app itself.
    - Fine-grained access control via IAM (tags/policies), independent of Kubernetes RBAC.
    - Works alongside **Pod Identity** (Section 7) so the pod authenticates as itself when fetching secrets, without hardcoding credentials.

---

## 5. Load Balancers

### LoadBalancers Intro

Kubernetes originally had cloud-provider logic baked into the control plane; today that's a separate **cloud controller** (EKS runs it for you), plus the **AWS Load Balancer Controller** specifically for load balancing.

Understanding this stack starts with understanding what a Kubernetes **Service** does under the hood: it exposes a **NodePort** on every node in the cluster, whether or not that node is running a matching pod. If a request lands on a node without the pod, **kube-proxy** quietly reroutes it to a node that does have it.

> 💡 AWS recommends setting `externalTrafficPolicy` to only route to nodes actually running the pod — this saves the extra kube-proxy hop (which can otherwise cross AZs and add latency/cost).

**Two main patterns for exposing services:**

![[Pasted image 20260826143551.png]]

Other useful pieces:

- **External DNS** — an open-source controller that watches Services/Ingresses and automatically creates matching Route 53 records, no manual DNS management required.
- **Global Load Balancer** — sits outside any single region, splitting traffic across regional load balancers by geography or weighted percentage. Not yet managed by the AWS Load Balancer Controller itself.

### Gateway Ingress

Kubernetes **Ingress is not graduating to a stable API** — it's being succeeded by the **Gateway API**. Think of Gateway API as "Ingress v2": same core idea (route L7 traffic into the cluster), but far more flexible and protocol-aware.

Routing is split into composable pieces:

- **GatewayClass** — groups gateways by which controller manages them (like IngressClass).
- **Gateway** — the actual listener; points to a backend Service.
- **HTTPRoute / TLSRoute / TCPRoute / UDPRoute / GRPCRoute** — purpose-built route types instead of Ingress's one-size-fits-all HTTP host/path matching. (TLSRoute, for example, routes purely on the SNI hostname from the TLS handshake, without terminating TLS itself.)

> ⚠️ The AWS Load Balancer Controller does **not** (as of this writing) support Gateway API resources. For Gateway-native routing on AWS today, look at **VPC Lattice** instead.

### VPC Lattice

AWS's implementation for Gateway API traffic — a network-meshing service that sits transparently below/above your VPCs.

- Core abstraction: a **service network** — a group of service endpoints registered via **AWS Cloud Map** (conceptually similar to how kube-dns works inside a cluster) that can reach each other as long as **IAM allows it**, regardless of VPC or even AWS account.
- Can connect not just Kubernetes Services, but also **Lambda functions and EC2 instances** into the same mesh.
- **Trade-offs:**
    - Everything is IAM-gated — heavier reliance on IAM policy than typical network policies.
    - Provisioning a service network can take **5–10 minutes** — a jarring wait if you're used to Kubernetes resources appearing almost instantly.

> 💡 This is squarely an **advanced, large-enterprise** tool. If you're running one or two clusters, there are simpler ways to route traffic between services. Save Lattice for dozens/hundreds of VPCs and accounts.

---

## 6. Compute & Scaling

Every EKS cluster needs somewhere for pods to actually run. There are three main paths, and they solve different problems.

### Fargate

"Serverless" compute — no EC2 instance ever shows up in your account.

- A **mutating webhook** swaps in a special **Fargate scheduler** for matching pods; the standard Kubernetes scheduler has no concept of Fargate at all.
- The Fargate scheduler calls out to AWS, provisions a right-sized node just for that one pod (adding overhead for the kubelet and any sidecars), and binds the pod to it.
- **Limitations:**
    - **No DaemonSets** — must convert to sidecars, which multiplies resource usage at scale.
    - **No EBS volumes.**
    - **No dynamic EFS provisioning.**
- **Best for:** workloads that need real isolation/security boundaries, or core cluster services (metrics-server, autoscaler) that benefit from being insulated from the churn of regular node upgrades.

### EKS Node Groups

The classic, most common compute option — EC2 instances grouped and backed by an **Auto Scaling Group (ASG)**.

- **Unmanaged node groups** — you handle everything: the ASG, joining the kubelet to the cluster, IAM credentials.
- **Managed node groups** — AWS handles version upgrades for you. Trigger an upgrade, and EKS rolls new instances in via the launch template while draining old ones out.
- Custom AMIs are supported in managed node groups too (via launch templates) — this used to be the main reason people stuck with unmanaged groups, but it's no longer necessary.
- **Expect churn during upgrades** — workloads often get rescheduled 3–5 times as they bounce between old and new instances before finally landing on a fully-upgraded node.
- You can run **multiple node groups** simultaneously (e.g. general compute + GPU node group).

### Karpenter

The newest way to get compute into a cluster — built by AWS, donated to the Kubernetes autoscaling SIG.

- Fundamentally different approach: **no pre-declared node groups**. When a pod is unschedulable, Karpenter looks at every EC2 instance type available in your region (not just what's in a pre-defined group), finds the **cheapest one that satisfies the pod's requirements**, and provisions it on demand.
- **Consolidation** — continuously re-evaluates whether it can bin-pack workloads onto fewer or cheaper nodes and swaps accordingly, to keep the cluster cost-efficient.
- Manages its own upgrade process by draining and replacing (or consolidating away) nodes as newer versions become available.

> ⚠️ **The catch:** your workloads need to be mature. Without **PodDisruptionBudgets**, **topology spread constraints**, and proper **resource requests** defined, Karpenter will happily schedule everything onto a single giant (cheapest) instance and reshuffle nodes aggressively — a great way to cause an outage if those guardrails aren't in place.

**Quick comparison:**

![[Pasted image 20260826143617.png]]

---

## 7. Redundancy & Resiliency

### Cluster Access

When you run `kubectl get nodes`, the request goes through an AWS-managed Application Load Balancer in front of the control plane. Your kubeconfig invokes the **aws-iam-authenticator** plugin to fetch AWS credentials and present them to that ALB. But AWS IAM and Kubernetes RBAC are two separate systems — something has to map one to the other:

- **aws-auth ConfigMap (legacy)** — a ConfigMap living in `kube-system` that manually maps IAM ARNs to RBAC roles. Plain text, so a formatting mistake could lock you out entirely; whoever created the cluster was always silently granted permanent cluster-admin with no clean way to revoke it.
- **EKS Access Entries API (current)** — the modern replacement. Access mappings are managed declaratively through the EKS API itself. Cluster creation and cluster administration can be cleanly split across different teams, and it's much safer to automate.

> 💡 aws-auth still works and still exists — but for a new cluster today, use the EKS Access Entries API instead.

### IRSA (IAM Roles for Service Accounts)

The first AWS-native solution for giving a **pod** (not a human) permission to call AWS APIs.

- Relies on the cluster's **OIDC endpoint**: a pod fetches a JWT token, exchanges it with **STS** for temporary credentials, and STS validates the request against an IAM trust policy tied to that OIDC provider.
- Works even outside EKS (self-managed control planes).
- **Real limitations:**
    - Hard cap of roughly **100 OIDC providers per AWS account**.
    - IAM trust relationships can only be reused **~5 times per role**.
    - You can't pre-create trust relationships until the cluster (and its OIDC URL) already exists.

### Pod Identity

The current, recommended approach — sometimes called "IRSA v2."

- The mapping between a **ServiceAccount** and an **IAM role** is stored directly in the **EKS API** — no OIDC, no STS `AssumeRoleWithWebIdentity` needed.
- A **DaemonSet** on each node hands out credentials locally via a local `169.254` endpoint.
- Because AWS trusts the EKS service principal itself, the **same IAM role can be reused everywhere** — access is scoped using tag-based (**ABAC**) policies instead of juggling dozens of near-identical roles.
- Only works on EKS itself (associations live in the EKS API) — not for self-managed Kubernetes control planes.

> 💡 IRSA and Pod Identity can run **side-by-side** during a migration — the injection webhook simply prefers Pod Identity when both are enabled for a workload.

### SG (Security Groups) for Pods

By default, every ENI on a node — and therefore every pod scheduled to it — **shares the same security group**. AWS does offer a way to assign distinct security groups per pod, using a private-API VPC controller that attaches special "trunk" ENIs.

**Real trade-offs:**

- Only some EC2 instance types support trunk ports, and there's no simple API to check — you have to consult a lookup table.
- Pods using trunk-port security groups aren't counted the same way in the VPC CNI's max-pods calculation, requiring manual node capacity adjustments.
- Combining this with a non-default CNI (Calico, Cilium) multiplies the complexity dramatically.

> 💡 **Recommended alternative stack:** Pod Identity for AWS-level access control, Network Policies for coarse IP/CIDR traffic rules, and separate node groups (or clusters) when you need a hard security boundary. Security groups for pods is a real feature — reach for it only if you already have deep in-house security-group tooling and expertise.

---

## 8. Upgrades and Maintenance

### EKS Monitoring

Almost everything eventually lands in **CloudWatch**.

- **Control plane logging** — a simple opt-in checkbox at cluster-creation time; API server / scheduler / controller-manager logs flow into a CloudWatch log group automatically.
- **CloudWatch Observability add-on** — installs the CloudWatch agent as a DaemonSet, enriching node and container logs with extra metrics.
- **ADOT (AWS Distro for OpenTelemetry)** — builds on the agent to export metrics/logs/traces to any OpenTelemetry-compatible backend, not just CloudWatch.
- **AWS X-Ray add-on** — builds on ADOT specifically for distributed tracing.
- **Fargate nodes** are the exception — since they can't run DaemonSets, logging instead uses a managed **Fluent Bit ("Firelens")** integration, triggered by creating an `aws-observability` namespace and a logging ConfigMap.
- **AMP (Managed Prometheus) & AMG (Managed Grafana)** — fully managed versions of the same open-source stacks, with IAM baked in for auth/access control, for teams that don't want to build every dashboard in CloudWatch.

### Upgrade Cycles

- A standard EKS cluster version is only supported for **14 months**.
- Kubernetes upstream ships **3 releases a year**, and EKS stays in lockstep — meaning you should expect to upgrade roughly **every 3–4 months** to stay comfortably ahead of the deadline.
- Miss the window and you move into **Extended Support**: another 12 months of life, but at roughly **5x the hourly cost** (~$0.10/hr standard → ~$0.60/hr extended).
- **AWS Upgrade Insights** (console or CLI) shows which deprecated or soon-to-be-removed APIs your cluster is actively using — genuinely useful, since the team running the cluster is often not the team that owns the workloads calling those APIs.

### EKS Upgrades

**In-place vs. blue/green:**

- **In-place (recommended, almost always):** DNS caching, load balancer cutover, and the number of teams depending on a stable environment make blue/green migrations take far longer and cost more (running two clusters at once) than expected.
- **Blue/green:** reserve for two specific situations — swapping out your CNI provider entirely, or jumping several major versions at once (in-place would otherwise force you through every version one at a time).

**An in-place upgrade always follows the same order:**

1. **Control plane upgrades first** — rolling in a new API server / controller-manager / scheduler behind the load balancer (fully managed by AWS; etcd migrates automatically too).
2. **Data plane upgrades next:**
    - Managed node groups roll one (or N, or a %) node at a time via the ASG.
    - Karpenter intelligently drains and replaces (or consolidates away) nodes as needed.
    - Fargate nodes are replaced by forcing a new Deployment rollout.
3. **Add-ons upgraded individually**, per add-on, via their own API calls — this doesn't happen automatically.

**Helpful tools:**

- **EKS cluster-insights API** — flags Kubernetes resources using APIs about to be removed (e.g. PodSecurityPolicies were removed between 1.24 and 1.25).
- **kube-no-trouble ("kubent")** — does a similar job with more workload-level detail, at the cost of needing broader RBAC/IAM permissions to inspect every namespace.

### EKS Addon

EKS add-ons are a distinctly **EKS-specific** concept (not a general Kubernetes one) — a way to install and version-manage baseline cluster services (VPC CNI, CoreDNS, EBS CSI driver, etc.) through the EKS API/console instead of a separate Helm install.

**Honest take:** they're still a bit rough. If you're upgrading a cluster, you end up upgrading the control plane, then your node groups, then separately upgrading every single add-on — with no single, holistic "test this all together" workflow.

> 💡 **Recommendation:** own your own baseline YAML — plain manifests, Helm charts you manage yourself, or a GitOps pipeline — rather than delegating your cluster's foundational services to AWS's add-on marketplace or a third-party vendor's release cadence. The more external dependencies stand between you and an upgrade, the harder every one of those 3–4-times-a-year upgrades becomes.

---

_Notes compiled from the KodeKloud AWS EKS course material._