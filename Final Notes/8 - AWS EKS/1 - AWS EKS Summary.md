# EKS Notes

A study guide to Amazon Elastic Kubernetes Service (EKS), organized from fundamentals through networking, storage, secrets, load balancing, compute, resiliency, and cluster maintenance.

Where a topic is a **feature or fix** (not just a definition), notes follow this structure:

> **Problem Statement** → what's broken/painful without it **Solution / What it fixes** → what the feature actually does **Analogy** → a simple real-world comparison

---

# 1. EKS Fundamentals

### What is EKS

EKS (Elastic Kubernetes Service) is AWS's **hosted/managed service for Kubernetes**. Every Kubernetes cluster is made of two halves:

- **Control plane** — etcd, the API server, the scheduler, the controller manager. The "brain" of the cluster.
- **Data plane** — the worker nodes that actually run your pods.

**Problem Statement** Running Kubernetes yourself means owning _everything_, including the scariest part — **etcd**, the distributed database behind every cluster. If etcd is lost, the cluster is effectively gone. You're on the hook for etcd backups, control-plane high availability, patching, and 24/7 uptime of the API server itself, on top of everything else you actually wanted to build.

**Solution / What it fixes** EKS draws a clean line down the middle of the cluster: AWS owns and operates the **control plane** (in an AWS-owned account — you just see "an EKS cluster" in the console), while you own the **data plane** (nodes, VPC, security groups) in your own account. The two sides are bridged by cross-account **Elastic Network Interfaces (ENIs)**.

**Analogy** Think of it like renting an apartment in a managed building vs. buying a standalone house. In the house (self-managed Kubernetes), you're responsible for the plumbing, the electrical, the roof — everything. In the managed apartment (EKS), the building owner handles the structural stuff (control plane) while you're still responsible for what's inside your unit (nodes, workloads). You get to focus on living there, not maintaining the foundation.

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

**Problem Statement** A single control plane running in one place is a single point of failure — if that one data center has an outage, your entire cluster (and every app running on it) goes down with it.

**Solution / What it fixes** EKS is a **regional service**, and a region is really a group of Availability Zones (separate data centers). Control plane pieces are spread across a **minimum of three AZs**, and etcd needs quorum, so it always keeps at least three copies alive and runs leader election if one goes down.

**Analogy** It's like a company keeping three backup copies of its most important documents in three different buildings across town, instead of one filing cabinet in one office. If one building burns down, the other two still have a valid, agreed-upon copy, and the business keeps running.

Beyond the standard control plane pieces, EKS also sets up a few extras:

- **OIDC endpoint** — authentication for anything that needs to call into AWS from inside the cluster.
- **CloudWatch logging** — opt-in, but the plumbing is already there.
- **aws-auth / EKS Access Entries API** — maps AWS IAM identities to Kubernetes RBAC.
- **Node Groups & Add-ons APIs** — technically part of the EKS control-plane API, but they create resources inside _your_ account.

### Deployment Options

There's no single right way to create a cluster — it comes down to what your team already knows:

- **Console** — works, but is consistently the most-complained-about experience, since EKS touches so many other AWS services (IAM, VPC, load balancers) that you end up bouncing between tabs.
- **CloudFormation** — hand-written YAML, or generated via the CDK. Good if you're an AWS-native shop already.
- **Terraform** — probably the most popular option. AWS maintains a set of community modules called **EKS Blueprints**. You own your own state file and access since it's not a managed service.
- **Everything else** — Pulumi, Cluster API, and a dozen other tools that ultimately script the same CloudFormation/Terraform/CLI calls for you.

> 💡 Whatever tool you pick, you'll need fairly broad IAM permissions to create a cluster — EKS touches EC2, VPCs, security groups, load balancers, and IAM roles all at once.

### Tools needed for EKS

- **kubectl** — the standard Kubernetes CLI.
- **aws-iam-authenticator** — a required `kubectl` plugin.

**Problem Statement** The EKS API server sits behind a load balancer that requires AWS-signed credentials — but `kubectl` natively only knows how to talk to a plain Kubernetes API using a kubeconfig token. Without something bridging the two, `kubectl` has no way to prove "this is really an authenticated AWS IAM user" to that load balancer.

**Solution / What it fixes** The **aws-iam-authenticator** plugin turns your IAM identity into a token `kubectl` can present, so the request is accepted.

**Analogy** It's like using your work badge (IAM identity) to get through a building's front security gate (the ALB) before you're even allowed to walk up to the receptionist's desk (the Kubernetes API) and ask for something.

- **eksctl / Terraform / CloudFormation / CDK** — whichever infrastructure tool you choose for creating and managing the cluster (see Deployment Options above).
- **AWS CLI** — for scripting direct API/CLI calls when you want more granular control than a wrapper tool provides.

### Networking (overview)

Every node and pod pulls its IP address from your VPC's subnets, using Elastic Network Interfaces (ENIs) to plug into the network. _(Full details in Section 2 — EKS Networking.)_

### Authentication (overview)

AWS IAM identities and Kubernetes RBAC are two separate systems — something has to map one to the other. _(Full details in Section 7 — Cluster Access.)_

---

# 2. EKS Networking

### How networking works

Every node and pod gets its IP address from the subnets in your VPC — and because subnets are bound to a single Availability Zone, so is every node and pod inside them. Nodes reach the network through **Elastic Network Interfaces (ENIs)**, which behave just like a physical network card plugged into a server.

**Problem Statement** EC2 instance types have hard limits on how many ENIs they can attach and how many IP addresses each ENI can hold (e.g. a small instance might support only 4–5 ENIs at 4–5 IPs each). Once you've scheduled enough pods to exhaust that pool, the VPC CNI has to register and attach an entirely new ENI — which is **not instant** (10–15 seconds or more), especially if the subnet is running low on free addresses and has to wait for IP leases to expire. Meanwhile, pods sit `Pending`.

**Solution / What it fixes** Two settings help the CNI stay ahead of that churn:

- **`WARM_ENI_TARGET`** — keep N extra ENIs pre-attached and ready before you need them.
- **`WARM_IP_TARGET`** — keep N extra IP addresses ready regardless of how many ENIs that takes.

**Analogy** It's like a coffee shop pre-brewing a few extra pots before the morning rush, instead of only starting a new pot once the current one runs completely dry. You avoid the customer standing there waiting for the next batch to finish.

### Prefix Delegation
![[Pasted image 20260826154236.png]]
**Problem Statement** Every EC2 instance type has hard limits on how many ENIs it can attach and how many IP addresses each ENI can hold. The VPC CNI requests **one IP address at a time**, so those limits fill up fast — even when the node still has plenty of spare CPU/memory. Once the IP pool is exhausted, attaching a whole new ENI takes 10–15+ seconds, and pods sit `Pending` purely because of IP scarcity, not compute.

**Solution / What it fixes** Instead of handing out one IP per ENI slot, prefix delegation assigns a whole **/28 block (16 addresses)** as a single route to one ENI slot.

- One ENI slot now unlocks **16 pod IPs** instead of one.
- A small instance that used to max out at ~4 usable pod IPs can jump to ~64.
- Enable it with the **`ENABLE_PREFIX_DELEGATION`** environment variable on the VPC CNI.

**Analogy** Think of the ENI as a parking garage entrance, and each IP address as a parking space. Without prefix delegation, every car (pod) that shows up needs the attendant to call the city and get one individual permit issued — and once the entrance is full, you have to build a whole new entrance just for a few more cars. With prefix delegation, the attendant goes to the city once and grabs a whole block of 16 pre-approved permits up front, so new cars just get handed a permit from the stack — no more calling the city for every single car.

SAMPLE SITUATION IN ENABLING PREFIX-DELEGATION
1. Open the `aws-k8s-cni.yaml` file in a text editor of your choice.
2. Locate the environment variable section for `ENABLE_PREFIX_DELEGATION`.
3. Add the following environment variable to enable prefix assignment: (DaemonSet)
```yaml
env:
- name: ENABLE_PREFIX_DELEGATION
  value: "true"
```

Ensure the environment variable is added correctly within the container spec section.
### IPv6

**Problem Statement** Even with prefix delegation, IPv4 address space inside a VPC is finite, and running out of IPs was historically the #1 scalability problem EKS customers hit — teams would add more subnets over and over and still eventually run out.

**Solution / What it fixes** On a dual-stack (IPv4 + IPv6) VPC, each node gets an **/80 block** — **100 trillion addresses** (more than the _entire_ IPv4 internet, which only has ~4 billion addresses total). Pods only get an IPv6 address; a local `169.254.x.x` address on the node handles NAT translation for the rare case a pod still needs to call a legacy IPv4 endpoint.

**Analogy** Switching from IPv4 to IPv6 in a VPC is like upgrading from a small town's local phone-number system (limited digits, numbers eventually run out and get reused) to a global numbering scheme so vast that literally every device on Earth, many times over, could have its own permanently unique number — you simply stop worrying about running out.

> ⚠️ **Watch out for:** many pods on one node all sharing that single outbound IPv4 NAT path — this can become a bottleneck if your surrounding infrastructure is still largely IPv4.

**Role of the 169.254 address**
![[Pasted image 20260826160004.png]]
**Problem Statement** Pods only get an IPv6 address — there's no dual-stack at the pod level. But not everything a pod needs to reach is IPv6-ready yet (some external endpoints or AWS resources may still be IPv4-only). Without some translation mechanism, an IPv6-only pod would have no way to reach an IPv4-only destination.

**Solution / What it fixes** The node sets up a local `169.254.x.x` address (a link-local, non-routable address — the same range used for things like the EC2 metadata endpoint). When a pod's traffic is headed to an IPv4 destination, it routes through this local address, which performs **source NAT** — translating the outbound request from IPv6 to IPv4, sending it out, and translating the response back to IPv6 on the way in. The pod never needs its own IPv4 address; the node handles the translation transparently.

**Analogy** The pod only speaks the newer language (IPv6), and some recipients only understand the older one (IPv4). The `169.254` address is a local, in-house interpreter sitting right next to the pod — every time the pod talks to an IPv4-only recipient, the interpreter translates it on the spot, sends it out, and translates the reply back. The pod never has to learn the old language itself.

### Network Policies

**Problem Statement** By default, any pod in a cluster can talk to any other pod. Without some way to restrict that, a compromised or misconfigured pod could reach far more of your infrastructure than it should — and locking traffic down usually means filing a ticket with a separate network/firewall team every time an app's dependencies change.

**Solution / What it fixes** Network Policies are the standard Kubernetes way to control which pods can talk to which other pods (ingress and egress). The AWS VPC CNI implements them using **eBPF** — a small program running in the node's Linux kernel, not a sidecar — so application teams can manage their own traffic rules as part of the same manifests they already deploy.

**Analogy** It's like giving each apartment in a building its own smart lock that the tenant controls, instead of every request to change access going through a single building-wide security office. The tenant (application team) can grant or restrict access to their own door as their needs change, without waiting on someone else.

> Different CNI providers (e.g. Calico) implement network policy differently — behavior may differ if you're not using the default VPC CNI.

SAMPLE SITUATIONS IN APPLYING

To see if the Network Policy is enabled run the command:
```bash
k get daemonset -n kube-system aws-ned -o yaml | less
```

Edit `amazon-vpc-cni` ConfigMap in the same file as above
```YAML
apiVersion: v1
kind: ConfigMap
metadata:
  name: amazon-vpc-cni
  namespace: kube-system
  labels:
    app.kubernetes.io/name: aws-node
    app.kubernetes.io/instance: aws-vpc-cni
    k8s-app: aws-node
    app.kubernetes.io/version: "v1.20.1"
data:
  enable-windows-ipam: "false"
  enable-network-policy-controller: "true" # Change value from false to true 
```

Edit `aws-node` DaemonSet arg as following:
```YAML
args:
            - --enable-ipv6=false
            - --enable-network-policy=true # Change from false to true
            - --enable-cloudwatch-logs=false
            - --enable-policy-event-logs=false
```



---

# 3. EKS Storage

**Problem Statement** Kubernetes constructs like `emptyDir` are fine for scratch space, but the data disappears the moment the pod does. Anything durable (like a database in a StatefulSet) needs storage that survives outside the pod's lifecycle entirely.

**Solution / What it fixes** AWS provides durable storage backends (EBS, EFS, and others) wired into the cluster through a **CSI driver** (Container Storage Interface) — a controller Deployment that manages volume lifecycle against the AWS API, plus a DaemonSet on every node that performs the actual mount.

**Analogy** `emptyDir` is like writing notes on a whiteboard in a conference room that gets erased the moment the meeting ends. EBS/EFS is like saving that same content to a shared drive — the meeting (pod) can end, but the notes (data) are still there afterward.
![[Pasted image 20260826180746.png]]
### EKS EBS (Elastic Block Store)

Block storage — like plugging in a new hard drive.
![[Pasted image 20260826175658.png]]
![[Pasted image 20260826180305.png]]

**Problem Statement** EBS volumes are **zone-bound**. If your autoscaler creates a replacement node in a different AZ than the volume lives in, the pod sits unschedulable while the autoscaler cycles through AZs trying to find the right one — potentially minutes of downtime for something like a database.

**Solution / What it fixes** This isn't really "fixed" so much as understood and planned around — pin storage-dependent workloads' scheduling to be AZ-aware, or use EFS instead if you need cross-AZ flexibility (see below).

**Analogy** An EBS volume is like a filing cabinet bolted to the floor of one specific office (AZ). If the employee who uses it gets reassigned to a different office building, they can't just carry the cabinet with them — someone has to notice and route them back to the _same_ building.

- 1 pod ↔ 1 volume (single-writer). Very low latency (same-AZ).
- Multiple volume types (GP2/GP3/IO) with different throughput/cost trade-offs; supports snapshots.
- Great fit for databases and other single-writer workloads.
![[Pasted image 20260826181641.png]]
![[Pasted image 20260826181446.png]]


#### STATIC PROVISIONING

Create an EBS volume using AWS CLI:
```
aws ec2 create-volume --size 10 --region us-east-1 --availability-zone us-east-1
```
Output:
```bash
{
    "AvailabilityZone": "us-east-1a",
    "CreateTime": "2026-08-26T10:48:12.000Z",
    "Encrypted": false,
    "Size": 10,
    "SnapshotId": "",
    "State": "creating",
    "VolumeId": "vol-0abcd1234ef567890",  # get this ID for the PV
    "Iops": 100,
    "Tags": [],
    "VolumeType": "gp2",
    "MultiAttachEnabled": false
}

```

Get the volume ID from the output and replace in the following YAML
```yaml
cat <<EOF | kubectl apply -f -
# Create a PersistentVolume (PV)
apiVersion: v1
kind: PersistentVolume
metadata:
  name: ebs-pv
spec:
  capacity:
    storage: 10Gi
  volumeMode: Filesystem
  accessModes:
    - ReadWriteOnce
  csi:
    driver: ebs.csi.aws.com
    fsType: ext4
    volumeHandle: vol-0abcd1234ef567890 # insert here the Volume-ID from the AWS command
  nodeAffinity:
    required:
      nodeSelectorTerms:
        - matchExpressions:
            - key: topology.kubernetes.io/zone
              operator: In
              values:
                - us-east-1a
---
# Create a PersistentVolumeClaim (PVC)
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: ebs-pvc
spec:
  storageClassName: ""
  volumeName: ebs-pv
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
---
# Create a pod using the PVC
apiVersion: v1
kind: Pod
metadata:
  name: ebs-pod
spec:
  nodeSelector: 
    topology.kubernetes.io/zone: us-east-1a
  containers:
  - name: app
    image: busybox
    command: [ "sh", "-c", "echo Hello Kubernetes! && sleep 3600" ]
    volumeMounts:
    - mountPath: "/data"
      name: ebs-storage
  volumes:
  - name: ebs-storage
    persistentVolumeClaim:
      claimName: ebs-pvc
EOF
```

#### DYNAMIC PROVISIONING 
Create a default storage class for dynamic provisioning.
```
cat <<EOF | kubectl apply -f -
# Create a StorageClass
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: ebs-sc
provisioner: ebs.csi.aws.com
volumeBindingMode: WaitForFirstConsumer
EOF
```

Create new PVC to use the default storage class and deploy new pod.
```
cat <<EOF | kubectl apply -f -
# PVC
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: ebs-pvc-new
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: ebs-sc
  resources:
    requests:
      storage: 10Gi
---
# Re-deploy pod
apiVersion: v1
kind: Pod
metadata:
  name: ebs-pod-new
spec:
  containers:
  - name: app
    image: busybox
    command: [ "sh", "-c", "echo Hello Kubernetes! && sleep 3600" ]
    volumeMounts:
    - mountPath: "/data"
      name: ebs-storage
  volumes:
  - name: ebs-storage
    persistentVolumeClaim:
      claimName: ebs-pvc-new
EOF
```

### EKS EFS (Elastic File System)
![[Pasted image 20260827093213.png]]
File storage (NFS) — "just give me files," no formatting or disk management needed.

**Problem Statement** Some workloads need multiple pods, possibly in different AZs, to read and write the _same_ files at the same time — something a zone-bound EBS volume simply cannot do.

**Solution / What it fixes** EFS is a **regional**, not zonal, NFS share — many pods across many AZs can mount the same volume simultaneously.

**Analogy** If EBS is a filing cabinet bolted to one office, EFS is a shared cloud drive that every office in every city can open and edit at once. Convenient — but just like any shared drive, if two people try to edit the same file at the same second, you can run into locking/conflict issues.

- Needs IAM + security group setup before it can be mounted.
- **Dynamic provisioning is not available on Fargate or Windows nodes.**
- Can be shared between zones
- Created outside the cluster

### Difference between EKS EBS and EKS EFS
Both EBS and EFS can back persistent storage for Kubernetes workloads on EKS, but they work very differently underneath. Here's the breakdown:

#### Core difference

**EBS (Elastic Block Store)** — block storage, like a virtual hard drive attached to a single EC2 instance.  
**EFS (Elastic File System)** — network file storage (NFS), like a shared network drive that multiple instances/pods can access at once.

#### Key comparisons
![[Pasted image 20260827093820.png]]

#### Practical rule of thumb

- **Need a fast dedicated disk for one pod (e.g., a database)?** → EBS
- **Need multiple pods across AZs to read/write the same files (e.g., shared media uploads, shared logs, WordPress `wp-content`)?** → EFS

One more nuance: EBS volumes are AZ-locked, which is a common gotcha in EKS — if your StatefulSet pod gets rescheduled to a node in a different AZ, it can get stuck waiting for a volume it can't attach to. EFS avoids that entirely since it's accessible from any AZ in the region, at the cost of higher per-operation latency compared to EBS.

### EKS Other Storage

- **FSx for Lustre** — AWS-managed Lustre, for high-throughput HPC-style workloads.
- **S3** — many apps talk to S3 natively via SDK. There's also a newer **S3 Mount Point CSI driver** that presents a bucket as a filesystem — handy, but **not POSIX-compliant** (renames, `mkdir`, etc. behave differently than a real filesystem).
- **Local volumes (NVMe)** — the fastest storage available in AWS, but fully ephemeral; data disappears when the node does.
- **In-cluster storage** — Longhorn, Rook/Ceph, OpenEBS, etc. More portable across clouds/on-prem, but more operational burden (you own the replication, backups, and failure handling).
![[Pasted image 20260827095322.png]]
![[Pasted image 20260827095307.png]]





---

# 4. EKS Secrets

### EKS Secrets Intro

**Problem Statement** Applications need sensitive values (API keys, passwords, tokens) available at runtime — but you don't want that sensitive data baked directly into container images or plain-text config files.

**Solution / What it fixes** A **Kubernetes Secret** looks like a ConfigMap, except the value is **base64-encoded** before being sent to the API server, keeping it separate from application code and config.

> ⚠️ **Important:** base64 is not encryption. It's trivially reversible — it obscures a value from a casual glance, but does **not** protect it. Anyone with API/etcd access (or the right RBAC) can decode it instantly.

**Analogy** Base64-encoding a secret is like writing a note in Pig Latin and putting it in an unlocked drawer. It _looks_ like it's hidden from a casual glance, but anyone who knows the trick (and it's a very simple, well-known trick) can read it in seconds. It's obscurity, not a lock.

### Kubernetes Secrets Options

**Problem Statement** Because native Secrets aren't actually encrypted, genuinely sensitive values (root passwords, third-party API keys) need real protection — encryption at rest, rotation, and fine-grained access control that Kubernetes RBAC alone doesn't provide.

**Solution / What it fixes** Store secrets outside the cluster in **AWS Secrets Manager** (or **HashiCorp Vault** for multi-cloud/on-prem), then mount them into pods using the **Secrets Store CSI Driver** — which can mount secrets as files, or sync them as short-lived Kubernetes Secrets that get created/deleted alongside the pod's lifecycle. This also enables automatic rotation, so a workload always reads the current value without needing AWS SDK credentials baked into the app.

**Analogy** It's like the difference between keeping your house key under the doormat (a native Secret — technically "hidden" but not really secure) versus using a proper safe with a combination that only certain trusted people know, and that gets changed on a schedule (Secrets Manager + Secrets Store CSI Driver). The doormat is convenient; the safe is what you actually want for anything valuable.

![[Pasted image 20260827130452.png]]

#### INTEGRATING AWS SECRET STORE TO A CLUSTER

```bash
# ============================================
# STEP 1: Install Secrets Store CSI Driver
# ============================================
helm repo add secrets-store-csi-driver https://kubernetes-sigs.github.io/secrets-store-csi-driver/charts
helm repo update

helm install csi-secrets-store secrets-store-csi-driver/secrets-store-csi-driver \
  --namespace kube-system \
  --set syncSecret.enabled=true   # enables syncing to native K8s Secrets (needed for secretObjects/envFrom)

# ============================================
# STEP 2: Install AWS provider (ASCP)
# ============================================
kubectl apply -f https://raw.githubusercontent.com/aws/secrets-store-csi-driver-provider-aws/main/deployment/aws-provider-installer.yaml

# Verify both are running (should see pods on every node, since they're DaemonSets)
kubectl get pods -n kube-system -l app=secrets-store-csi-driver
kubectl get pods -n kube-system -l app=csi-secrets-store-provider-aws


# ============================================
# STEP 3: Create IAM policy (read access to the secret)
# ============================================
aws iam create-policy \
  --policy-name eks-secrets-access \
  --policy-document '{
    "Version": "2012-10-17",
    "Statement": [{
      "Effect": "Allow",
      "Action": ["secretsmanager:GetSecretValue", "secretsmanager:DescribeSecret"],
      "Resource": "arn:aws:secretsmanager:REGION:ACCOUNT_ID:secret:your-secret-name-*"
    }]
  }'

# Expected Output:====================================================
{
    "Policy": {
        "PolicyName": "eks-secrets-access",
        "PolicyId": "ANPAJ2UCCR6DPCEXAMPLE",
        "Arn": "arn:aws:iam::123456789012:policy/eks-secrets-access",
        "Path": "/",
        "DefaultVersionId": "v1",
        "AttachmentCount": 0,
        "PermissionsBoundaryUsageCount": 0,
        "IsAttachable": true,
        "CreateDate": "2026-08-27T10:15:00Z",
        "UpdateDate": "2026-08-27T10:15:00Z"
    }
}
#=====================================================================
# NOTE: copy the returned "Arn" — needed for --attach-policy-arn in step 4

# ============================================
# STEP 4: Create IAM role + Kubernetes ServiceAccount (IRSA)
# ============================================
eksctl create iamserviceaccount \
  --name secrets-sa \
  --namespace default \
  --cluster your-cluster-name \
  --attach-policy-arn arn:aws:iam::ACCOUNT_ID:policy/eks-secrets-access \
  --approve
# NOTE: creates both the IAM role (trusted via OIDC) and the K8s ServiceAccount, linked together
# Expected Output: ===================================================
2026-08-27 10:16:01 [ℹ]  1 iamserviceaccount (default/secrets-sa) was included (based on the include/exclude rules)
2026-08-27 10:16:01 [!]  serviceaccounts that exist in Kubernetes will be excluded, use --override-existing-serviceaccounts to override
2026-08-27 10:16:01 [ℹ]  1 task: { create IAM role for serviceaccount "default/secrets-sa" }
2026-08-27 10:16:02 [ℹ]  building iamserviceaccount stack "eksctl-your-cluster-name-addon-iamserviceaccount-default-secrets-sa"
2026-08-27 10:16:02 [ℹ]  deploying stack "eksctl-your-cluster-name-addon-iamserviceaccount-default-secrets-sa"
2026-08-27 10:16:02 [ℹ]  waiting for CloudFormation stack "eksctl-your-cluster-name-addon-iamserviceaccount-default-secrets-sa"
2026-08-27 10:16:35 [ℹ]  waiting for CloudFormation stack "eksctl-your-cluster-name-addon-iamserviceaccount-default-secrets-sa"
2026-08-27 10:16:35 [ℹ]  created serviceaccount "default/secrets-sa"
#=====================================================================

# ============================================
# STEP 5: Verify ServiceAccount is IRSA-linked
# ============================================
kubectl get serviceaccount secrets-sa -n default -o yaml
# Should show annotation: eks.amazonaws.com/role-arn: arn:aws:iam::...
```

```yaml
# ============================================
# STEP 6: SecretProviderClass
# ============================================
apiVersion: secrets-store.csi.x-k8s.io/v1
kind: SecretProviderClass
metadata:
  name: aws-secrets
  namespace: default
spec:
  provider: aws
  parameters:
    objects: |
      - objectName: "your-secret-name"
        objectType: "secretsmanager"
  secretObjects:
    - secretName: my-app-secret
      type: Opaque
      data:
        - objectName: "your-secret-name"
          key: password
---
# ============================================
# STEP 7: Pod using the secret
# ============================================
apiVersion: v1
kind: Pod
metadata:
  name: my-app
  namespace: default
spec:
  serviceAccountName: secrets-sa
  containers:
    - name: app
      image: my-app:latest
      volumeMounts:
        - name: secrets-store
          mountPath: "/mnt/secrets"
          readOnly: true
      envFrom:
        - secretRef:
            name: my-app-secret
  volumes:
    - name: secrets-store
      csi:
        driver: secrets-store.csi.k8s.io
        readOnly: true
        volumeAttributes:
          secretProviderClass: "aws-secrets"
```

```bash
# ============================================
# STEP 8: Verify
# ============================================
kubectl exec my-app -- cat /mnt/secrets/your-secret-name
kubectl get secret my-app-secret -o yaml
```


---

# 5. Load Balancers

### LoadBalancers Intro

**Problem Statement** Pods are ephemeral — they get created, rescheduled, and destroyed constantly, and each one gets its own internal IP. External users (or other services) need a stable, well-known way to reach "the app," without caring which specific pod, node, or IP is currently serving it.

**Solution / What it fixes** A Kubernetes **Service** exposes a stable **NodePort** on every node in the cluster. If a request lands on a node without a matching pod, **kube-proxy** quietly reroutes it to a node that does have one. A `LoadBalancer`-type Service then puts an actual AWS ELB/NLB/ALB in front of that, giving external traffic one stable entry point regardless of what's happening to the pods behind it.

**Analogy** It's like a company's main reception phone number. Employees (pods) come and go, move desks, or are out sick — but callers only ever need to remember the one main number, and the receptionist (kube-proxy/load balancer) figures out who's actually available to route the call to right now.
![[Pasted image 20260827135019.png]]
![[Pasted image 20260827135153.png]]
> 💡 AWS recommends setting `externalTrafficPolicy` to only route to nodes actually running the pod — this saves the extra kube-proxy hop (which can otherwise cross AZs and add latency/cost).

**Two main patterns for exposing services:**
![[Pasted image 20260827131531.png]]

---
kube-proxy handles load balancing **inside** the cluster (node ↔ node, pod ↔ pod). A cloud Load Balancer handles traffic getting **into** the cluster from the outside world. They work together in layers:
```
Internet
   │
   ▼
[Cloud Load Balancer]  (AWS ELB/NLB — external, provisioned via Service type: LoadBalancer)
   │  sends traffic to any Node's IP on a NodePort
   ▼
[kube-proxy on that Node]  (internal — picks a healthy Pod from that Service)
   │
   ▼
[Pod]
```
#### Step by step, for `Service type: LoadBalancer`:
1. You create a Service with `type: LoadBalancer`.
2. The cloud provider (AWS, via the AWS Load Balancer Controller or in-tree provider) spins up a real **ELB/NLB**, pointed at your worker nodes.
3. That external LB sends traffic to **any node**, hitting a **NodePort** it opened.
4. **kube-proxy on that node** intercepts the traffic and forwards it to one of the correct Pod IPs — even if that Pod is running on a _different_ node — via cluster networking.
5. So even though the external LB just picked "a node," kube-proxy is what actually gets it to a live, healthy Pod.
---

Other useful pieces:
- **External DNS** — an open-source controller that watches Services/Ingresses and automatically creates matching Route 53 records.
- **Global Load Balancer** — sits outside any single region, splitting traffic across regional load balancers by geography or weighted percentage.

### Gateway Ingress
##### NGINX Ingress Controller
![[Pasted image 20260827143932.png]]
```
External Traffic → Load Balancer → NodePort → Ingress Controller → Service (port) → Pod
```
- **External Traffic** — request hits your domain (e.g. `myapp.fun`)
- **Load Balancer** — cloud LB (AWS ELB/NLB), spreads traffic across nodes
- **NodePort** — the 80/443 door opened on every node, owned by NGINX's Service
- **Ingress Controller** — NGINX pod reads Ingress Rules, matches host/path, decides which app
- **Service (port)** — stable internal address (e.g. `myapp-service:5000`), load-balances across healthy Pods
- **Pod** — your actual app container
##### AWS Load Balancer
![[Pasted image 20260827145216.png]]
###### IP target mode (recommended)
```
External Traffic → AWS ALB → Pod
```

```
Client → myapp.fun
       → AWS ALB (reads listener rules — auto-created by Load Balancer Controller
                   from your Ingress YAML — matches host/path)
       → Pod IP directly (target group points straight at pod IPs)
```

**Problem Statement** Kubernetes Ingress only really understands HTTP host/path routing — it has no clean, standardized way to route TCP, UDP, gRPC, or TLS-SNI-based traffic, and it's not even graduating to a stable Kubernetes API. Different Ingress controllers ended up inventing their own custom annotations to fill the gaps, making configs non-portable between controllers.

**Solution / What it fixes** The **Gateway API** is "Ingress v2" — the same core idea (route L7+ traffic into the cluster) but split into composable, purpose-built pieces: **GatewayClass** (which controller manages it), **Gateway** (the listener), and route types (**HTTPRoute, TLSRoute, TCPRoute, UDPRoute, GRPCRoute**) instead of one-size-fits-all HTTP matching.

**Analogy** Ingress is like a single mail slot that only accepts standard letters — if you need to send a package or a fragile item, you're stuck jamming it in sideways with proprietary tape (custom annotations). Gateway API is like a proper mail room with separate, purpose-built slots for letters, packages, and fragile items — each handled the right way, by design.

> ⚠️ The AWS Load Balancer Controller does **not** (as of this writing) support Gateway API resources. For Gateway-native routing on AWS today, look at **VPC Lattice** instead.

### VPC Lattice

**Problem Statement** As an organization grows to dozens or hundreds of VPCs and AWS accounts, connecting services across all of them with traditional VPC peering, routing tables, and security groups becomes a tangled, hard-to-manage mess — every new connection is manual networking work.

**Solution / What it fixes** VPC Lattice is a network-meshing service with a **service network** abstraction — a group of service endpoints registered via **AWS Cloud Map** that can reach each other as long as **IAM allows it**, regardless of VPC or even AWS account. It can connect Kubernetes Services, Lambda functions, and EC2 instances into the same mesh.

**Analogy** Traditional VPC networking across many accounts is like every department in a large company having to run its own physical phone line to every other department it needs to talk to — a nightmare of individual wiring. VPC Lattice is like giving everyone a company-wide directory and phone system: as long as you're authorized (IAM), you just dial the name and get connected, without anyone having to run new wires.

> 💡 This is squarely an **advanced, large-enterprise** tool — provisioning a service network can take 5–10 minutes, and everything is IAM-gated. Save it for dozens/hundreds of VPCs and accounts, not one or two clusters.

#### VPC Lattice — Summary

**What it is:** A fully managed AWS application networking service — think of it as AWS-native service-to-service connectivity + traffic management, built directly into AWS's network infrastructure (not something you run as pods).
##### The problem it solves
Plain `Ingress` (NGINX or ALB) only handles **north-south** traffic (external → single cluster) and is **scoped to one cluster**. It has no answer for:
- Services in **different EKS clusters** needing to talk to each other
- Services across **different VPCs or AWS accounts**
- The newer **Gateway API** standard (AWS Load Balancer Controller only supports old-style `Ingress`)
##### What it actually does
- Connects services across **multiple VPCs, accounts, and EKS clusters** — this is its core differentiator
- Implements the **Gateway API** on AWS (via the **AWS Gateway API Controller**)
- Handles both **north-south** (external traffic in) and **east-west** (service-to-service) communication
- Provides built-in **security** (defense-in-depth between services) and **observability** (traffic monitoring) — without needing to run a separate service mesh like Istio
##### How it fits the model you already know
```
Ingress/Gateway YAML (the spec)
        ↓
Gateway API Controller (reads it — this is VPC Lattice's controller)
        ↓
VPC Lattice Service Network (the actual infra — spans VPCs/clusters/accounts)
        ↓
Target Pods
```
Same relationship as NGINX/ALB — **the controller is not in the traffic path**, it just provisions and syncs the real infrastructure (VPC Lattice's service network) based on your `Gateway`/`HTTPRoute` YAML.

##### Role structure (Gateway API personas)

| Resource     | Owned by                                           |
| ------------ | -------------------------------------------------- |
| GatewayClass | Infra provider (defines "use VPC Lattice")         |
| Gateway      | Cluster operator (listener, TLS, namespace access) |
| HTTPRoute    | App developer (actual routing rules)               |

##### When to use it vs. NGINX/ALB

|Situation|Recommended|
|---|---|
|Single cluster, simple routing|NGINX or ALB — simpler|
|Need Gateway API on AWS specifically|VPC Lattice (only AWS-native option)|
|Multiple EKS clusters need to talk to each other|**VPC Lattice**|
|Multi-account/multi-VPC service mesh, no Istio|**VPC Lattice**|
|Just need external traffic → one app|Overkill — stick with ALB/NGINX|

**Bottom line:** VPC Lattice isn't a competitor to NGINX/ALB for basic ingress — it's solving a bigger, different problem (cross-cluster/cross-account service networking + Gateway API support). Most single-cluster setups don't need it; it shines once you're operating **multiple EKS clusters or AWS accounts** that need secure, observable service discovery between them.

##### Summary
All three ultimately _result in_ load balancing happening somewhere — but the actual load-balancing work happens in different places for each.

###### Where the real load balancing happens
![[Pasted image 20260827154639.png]]
###### The common thread
In every case, the Ingress/Gateway YAML is just the _instruction_ — it says "route requests matching X to Service Y." Something else has to take that instruction and actually **pick which of the (possibly many) backend Pods gets each individual request**. That "picking" step — spreading requests across multiple healthy targets — _is_ load balancing.

```
NGINX:       Ingress rules  →  NGINX picks a Pod (proxy load balancing)
ALB:         Ingress rules  →  ALB target group picks a Pod (native AWS LB)
VPC Lattice: Gateway/Route  →  Service network picks a target (native AWS LB, wider scope)
```
###### Two layers of load balancing, always
This connects back to what we covered earlier with kube-proxy — there are actually **two separate load-balancing decisions** happening in any of these setups:

1. **Which node does traffic land on first?** — handled by the cloud Load Balancer (or VPC Lattice's network) picking a node/target.
2. **Which Pod, of the app's replicas, actually gets the request?** — handled by whichever component owns that decision: NGINX (software), the ALB's target group (native AWS), or VPC Lattice's service (native AWS, cross-cluster).
###### Why this distinction matters practically
- **NGINX** = load balancing is a **workload** you run and pay compute for (the NGINX pods themselves consume CPU/memory, and you're responsible for scaling them).
- **ALB / VPC Lattice** = load balancing is **outsourced to AWS** — you don't run any proxy pods for it; AWS's managed infrastructure does the actual balancing, and you just pay for the managed service.

So to directly answer: **Ingress and Gateway API are the "rules for" load balancing, while NGINX, ALB, and VPC Lattice are the three different "engines" that actually perform it** — one running as software inside your cluster, the other two running as AWS's own managed network infrastructure outside it.

#### ==SAMPLE LAB:==
**[[LOAD BALANCING LAB]]**

---

# 6. Compute & Scaling

Every EKS cluster needs somewhere for pods to actually run — three main paths, each solving a different problem.

### Fargate

**Problem Statement** Managing EC2 instances — patching, right-sizing, capacity planning — is operational overhead some teams don't want at all, especially for workloads that need strict isolation (no "noisy neighbor" pods sharing the same node).

**Solution / What it fixes** Fargate is "serverless" compute — no EC2 instance ever shows up in your account. A mutating webhook swaps in a special Fargate scheduler for matching pods, which provisions a right-sized node just for that one pod and binds it there.

**Analogy** A regular EC2 node group is like renting a whole shared office floor and deciding how to arrange desks for your team yourself. Fargate is like booking a fully serviced, private single-person office on demand each time someone needs to work — no floor plan to manage, but you can't just walk over and share resources with a neighboring office (no DaemonSets, no shared EBS/EFS).

- **Limitations:** No DaemonSets (convert to sidecars), no EBS volumes, no dynamic EFS provisioning.
- **Best for:** isolated/security-sensitive workloads, or core cluster services (metrics-server, autoscaler) insulated from regular node-upgrade churn.

### EKS Node Groups
![[Pasted image 20260827173012.png]]
**Problem Statement** Someone needs to actually provision, join, and keep EC2 instances updated as Kubernetes nodes — and doing that entirely by hand (creating the ASG, bootstrapping the kubelet, managing IAM credentials, rolling AMI upgrades) is slow and error-prone.

**Solution / What it fixes** Node groups group EC2 instances behind an Auto Scaling Group. **Managed node groups** let AWS handle version upgrades for you — trigger an upgrade, and EKS rolls new instances in via the launch template while draining old ones out.

**Analogy** Unmanaged node groups are like personally hand-building and maintaining a fleet of company vehicles yourself. Managed node groups are like leasing that same fleet from a dealer who handles the maintenance schedule and swaps in updated models for you — you still decide how many vehicles (nodes) you need, but the mechanical upkeep is off your plate.

> Expect churn during upgrades — workloads often get rescheduled 3–5 times as they bounce between old and new instances.

### Karpenter
Karpenter is a **node autoscaler** — it watches for unschedulable Pods (Pods stuck `Pending` because no node has capacity) and automatically provisions new nodes to fit them. When nodes are no longer needed, it removes them too.

So functionally: yes, it scales nodes up and down based on demand — that's autoscaling.
#### How it's different from Cluster Autoscaler (the "traditional" way)
![[Pasted image 20260827174929.png]]

**Problem Statement** Pre-defining node groups requires guessing ahead of time every combination of instance type/size your workloads might need — and even then, unused capacity in those groups is wasted spend, while workloads that don't fit any existing group's shape simply can't be scheduled efficiently.

**Solution / What it fixes** Karpenter has no pre-declared node groups. When a pod is unschedulable, it looks at every EC2 instance type available in your region, finds the **cheapest one that satisfies the pod's requirements**, and provisions it on demand — then continuously "consolidates" (bin-packs or swaps nodes) to keep costs efficient.

**Analogy** Traditional node groups are like a taxi company that only owns a fixed set of pre-purchased car models and has to hope one of them roughly fits every passenger's needs. Karpenter is like an on-demand rideshare service that looks at exactly who needs a ride right now and summons the cheapest vehicle that actually fits them — and reshuffles/consolidates rides afterward to keep the whole fleet running efficiently.

==More on Consolidation==:
#### [[Karpenter Consolidation]]

> ⚠️ **The catch:** your workloads need to be mature (PodDisruptionBudgets, topology spread constraints, resource requests) or Karpenter's aggressive optimizing can cause real outages.


## **Quick comparison:**
![[Pasted image 20260827131611.png]]


## [[Compute Demo]]


---

# 7. Redundancy & Resiliency

### Cluster Access

**Problem Statement** AWS IAM and Kubernetes RBAC are two completely separate permission systems — something has to map "this AWS identity" to "this Kubernetes permission level." The original approach (a plain-text ConfigMap) meant a single formatting typo could lock everyone out, and whoever created the cluster was silently granted permanent, unrevokable cluster-admin.

**Solution / What it fixes** The **EKS Access Entries API** manages these mappings declaratively through the EKS API itself instead of a fragile in-cluster file — cluster creation and cluster administration can be cleanly split across teams, and it's much safer to automate.

**Analogy** The old aws-auth ConfigMap is like managing building access with a single shared handwritten sign-in sheet — one smudge or crossed-out line and the whole system is unreliable, and whoever built the building keeps a permanent master key with no way to take it back. The Access Entries API is like a proper digital badge system managed centrally by security — auditable, revocable, and far harder to break by accident.

> 💡 aws-auth still works and still exists — but for a new cluster today, use the EKS Access Entries API instead.

==More Information==: [[The Bridge Between AWS and Kubernetes Authentication]]
### IRSA (IAM Roles for Service Accounts)

**Problem Statement** A pod running inside a cluster often needs to call AWS APIs (e.g. read from S3) — but pods aren't EC2 instances, so they don't naturally have an IAM role the way a normal EC2-based application would. Handing every pod the _node's_ IAM role would over-grant permissions to everything running on that node.

**Solution / What it fixes** IRSA lets a pod exchange a JWT (fetched from the cluster's OIDC endpoint) with AWS STS for temporary, scoped IAM credentials — tied to a specific Kubernetes ServiceAccount, not the whole node.

>This literally means: _"Here's a web identity token (the JWT) — if it's valid and matches a trust policy, give me temporary credentials for this IAM role."_
```
1. Pod starts, using ServiceAccount "secrets-sa"
        ↓
2. Kubernetes mounts a JWT into the Pod
   (signed by the cluster's OIDC endpoint,
    claims: "I am secrets-sa in namespace default on cluster X")
        ↓
3. AWS SDK inside the Pod automatically calls:
   sts:AssumeRoleWithWebIdentity
   (sends the JWT + the target IAM role ARN)
        ↓
4. AWS STS checks the IAM role's trust policy:
   "Do you trust tokens issued by this cluster's OIDC provider,
    specifically for this ServiceAccount?"
        ↓
5. If it matches → STS issues temporary credentials
   (access key, secret key, session token — expires in ~1hr)
        ↓
6. Pod uses those temporary credentials to call
   AWS APIs (e.g., Secrets Manager, S3, DynamoDB)
```

**Analogy** Without IRSA, every employee (pod) sharing an office (node) would have to use the office's master key card (the node's IAM role) to get anywhere — way too much access for any one person. IRSA is like each employee getting their own individually scoped badge, checked at the door by a security desk (STS) that verifies who they say they are before issuing access.

> **Real limitations:** ~100 OIDC providers per AWS account, IAM trust relationships reusable only ~5x per role, and the OIDC URL isn't known until the cluster already exists.

==More Information==: [[IRSAIAM Roles for Service Accounts]]
### Pod Identity

**Problem Statement** IRSA's OIDC/STS handshake works, but its real-world limits (OIDC provider caps, IAM trust-relationship reuse limits, needing the cluster to exist before you can set up trust) become genuinely painful at scale, with many clusters and many workloads.

**Solution / What it fixes** Pod Identity stores the ServiceAccount ↔ IAM role mapping directly in the **EKS API** — no OIDC, no STS `AssumeRoleWithWebIdentity`. A DaemonSet on each node hands out credentials locally. Because AWS trusts the EKS service principal itself, the same IAM role can be reused everywhere, scoped with tag-based (ABAC) policies.
#### Runtime flow (when a Pod needs credentials)
```
1. Pod's SDK calls the local credentials endpoint:
   http://169.254.170.23/v1/credentials
        ↓
2. Pod Identity Agent (DaemonSet on the node) intercepts the call
        ↓
3. Agent calls the EKS Auth API:
   eks-auth:AssumeRoleForPodIdentity
        ↓
4. EKS itself checks: does an association exist for
   this Pod's namespace + ServiceAccount?
        ↓
5. If yes → temporary, scoped credentials returned to the Pod
```
>The key difference: the trust relationship is managed by EKS itself, not by an OIDC trust policy on the IAM role — meaning one IAM role can be reused across multiple clusters without modifying its trust policy.

**Analogy** If IRSA is a security desk that has to individually verify a signed letter of introduction (JWT) every single time, Pod Identity is like the building itself already having a pre-established, trusted relationship with the badge system — badges just work, everywhere in the building, without a separate verification letter each time.

> 💡 IRSA and Pod Identity can run **side-by-side** during a migration — the injection webhook simply prefers Pod Identity when both are enabled.


==More Information==: [[Pod Identity]]
### SG (Security Groups) for Pods

**Problem Statement** By default, every ENI on a node — and therefore every pod scheduled to it — shares the same security group. If one workload on a node needs access to a sensitive RDS instance, every other pod on that node technically has the same network-level path available (IAM still gates actual access, but the network door is open to all of them).

**Solution / What it fixes** AWS offers per-pod security groups using a private-API VPC controller that attaches special "trunk" ENIs, letting different pods on the same node carry different security groups.

**Analogy** Without this, it's like every apartment on a floor sharing one single building entrance keycard — anyone who lives on that floor could technically walk up to any door, even if they can't get inside without their apartment's own key (IAM). Security groups for pods is like giving each apartment its own dedicated building entrance — technically possible, but expensive and complicated to retrofit into an existing building (limited instance-type support, awkward pod-count accounting).

> 💡 **Recommended alternative stack:** Pod Identity for AWS-level access control, Network Policies for coarse traffic rules, and separate node groups/clusters for hard security boundaries. Reach for security groups on pods only with deep in-house expertise already in place.

---

# 8. Upgrades and Maintenance

### EKS Monitoring

**Problem Statement** When something goes wrong in a cluster — a crashing pod, a slow API server, a scheduler that isn't placing workloads — you need visibility into _what happened and when_, but that data doesn't just appear on its own; someone has to collect, ship, and store it.

**Solution / What it fixes** Control plane logging is a simple opt-in at cluster creation. On top of that, layered add-ons progressively enrich what you capture: the **CloudWatch Observability add-on** (agent as a DaemonSet), **ADOT** (exports to any OpenTelemetry backend), and **AWS X-Ray** (distributed tracing). Fargate nodes use a managed **Fluent Bit ("Firelens")** integration instead, since they can't run DaemonSets. **AMP/AMG** offer fully managed Prometheus/Grafana for teams that don't want to live entirely in CloudWatch.

**Analogy** Running a cluster with no monitoring is like driving a car with no dashboard — it might be running fine, or the engine might be about to fail, and you'd have no way to know until it actually breaks down. Each of these observability layers is another gauge on the dashboard: logs tell you what happened, metrics tell you the current state, and traces show you exactly which part of the trip took too long.

### Upgrade Cycles

**Problem Statement** Kubernetes upstream ships 3 releases a year and only supports each version for a limited window. If a cluster falls too far behind, it becomes both a security risk (unpatched, deprecated APIs) and, on EKS specifically, a much more expensive place to keep running.

**Solution / What it fixes** A standard EKS cluster version is supported for 14 months at the normal price (~$0.10/hr). Miss that window and you move into **Extended Support** — another 12 months of life, but at roughly 5x the hourly cost (~$0.60/hr). **AWS Upgrade Insights** flags which deprecated/soon-to-be-removed APIs your cluster is actively using, before you upgrade.

**Analogy** It's like a car's manufacturer warranty and parts-availability window. Keep up with routine maintenance (upgrades) within the covered window, and it's relatively cheap and low-drama. Let it lapse, and you're now paying premium prices for increasingly hard-to-source parts (extended support) — and eventually you have to do the work anyway.

### EKS Upgrades

**Problem Statement** Upgrading a live cluster risks disrupting every workload running on it — but standing up an entirely new cluster and migrating everything over (DNS, load balancers, all dependent teams) is slow and doubles your running costs in the meantime.

**Solution / What it fixes** **In-place upgrades** (recommended almost always) roll the control plane first, then node groups/Karpenter/Fargate, then add-ons individually — all without standing up a second cluster. **Blue/green** is reserved for two specific cases: swapping the CNI provider entirely, or jumping several major versions at once.

**Analogy** In-place upgrading is like renovating a house room by room while still living in it — some disruption, but you never have to move out. Blue/green is like building an entirely new house next door and moving everything over — cleaner in theory, but you're paying for two houses and coordinating a full move until it's done. You'd only do that for a full teardown-level change, not routine maintenance.

**Helpful tools:** the **EKS cluster-insights API** and **kube-no-trouble ("kubent")** both flag resources using APIs that are about to be removed, before you upgrade.

### EKS Addon

**Problem Statement** Baseline cluster services (VPC CNI, CoreDNS, EBS CSI driver) need to be installed and kept up to date somehow — doing it entirely by hand via Helm means one more thing to track and version yourself, cluster by cluster.

**Solution / What it fixes** EKS Add-ons let you install and version-manage these baseline services through the EKS API/console instead of a separate Helm install.

**Analogy** It sounds like the fix should be complete — like buying pre-assembled furniture instead of building it yourself. In practice, it's closer to pre-assembled furniture that still needs its _own_ separate instruction manual and toolset every time you want to update it: when you upgrade a cluster, you still upgrade the control plane, then node groups, then **individually** upgrade every single add-on — no single "test it all together" workflow yet.

> 💡 **Recommendation:** own your own baseline YAML — plain manifests, Helm charts you manage yourself, or a GitOps pipeline — rather than delegating your cluster's foundational services to AWS's add-on marketplace. The more external dependencies stand between you and an upgrade, the harder every one of those 3–4-times-a-year upgrades becomes.

---

_Notes compiled from the KodeKloud AWS EKS course material._