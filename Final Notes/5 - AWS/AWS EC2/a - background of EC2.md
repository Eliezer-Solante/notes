**Amazon EC2 (Elastic Compute Cloud)** is AWS's core service for renting virtual servers in the cloud. Instead of buying and maintaining physical hardware, you spin up "instances" — virtual machines — that you can use to run applications, host websites, process data, or basically anything you'd do with a physical server.
![[Pasted image 20260806124448.png]]
### Key concepts

**Instances**  
A single virtual server. You choose the OS (Linux, Windows, etc.), and it behaves like a regular computer you can SSH or RDP into.

**Instance Types**  
EC2 offers different hardware profiles optimized for different workloads:

- **General purpose** (e.g., `t3`, `m5`) — balanced CPU/memory, good default choice
- **Compute optimized** (`c5`) — high CPU, good for batch processing, gaming servers
- **Memory optimized** (`r5`) — high RAM, good for databases, caching
- **Storage optimized** (`i3`) — fast local storage, good for data warehousing
- **Accelerated computing** (`p4`, `g5`) — GPUs, good for ML training, graphics

**AMI (Amazon Machine Image)**  
A template that defines what's pre-installed on your instance — OS, software, configurations. You can use AWS-provided AMIs, community ones, or build your own.

**Pricing models**

- **On-Demand** — pay per second/hour, no commitment, most flexible, most expensive
- **Reserved Instances** — commit to 1-3 years for a big discount
- **Spot Instances** — bid on unused AWS capacity for steep discounts, but AWS can reclaim it with short notice
- **Savings Plans** — flexible commitment-based discount, similar to Reserved

**Elastic**  
The name refers to elasticity — you can scale instances up/down (bigger/smaller hardware) or out/in (more/fewer instances) based on demand, often automatically via **Auto Scaling Groups**.

### How it fits with the VPC diagram from before

EC2 instances are exactly the kind of resource that lives _inside_ a VPC's subnets. A web server EC2 instance might sit in a **public subnet** (reachable via an Internet Gateway), while a database EC2 instance sits in a **private subnet** (protected behind NACLs/Security Groups, only reachable internally).

![[Pasted image 20260806140211.png]]