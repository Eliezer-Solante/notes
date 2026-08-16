
![[Pasted image 20260806140353.png]]

This diagram shows the full **containment hierarchy** AWS uses to organize infrastructure — each layer nested inside the one before it, for both redundancy and security.

### Reading the diagram

**AWS Cloud → Region → Availability Zone → VPC → Subnet → Security Group → EC2**

- **Region**: a geographic area (e.g., "us-east-1"). Everything else lives inside one.
- **VPC**: your isolated network, drawn spanning _both_ Availability Zones — this is the key detail. A VPC isn't confined to one AZ; it stretches across multiple for redundancy.
- **Availability Zone (AZ)**: a physically separate data center within the region. The diagram shows **two AZs**, each with its own public and private subnet — this is what lets your app survive a data center outage.
- **Public/Private Subnet**: each AZ gets its own pair, mirroring the same public/private pattern we covered earlier, just duplicated for redundancy.
- **Security Group**: sits around the resource (EC2 instance) inside each subnet, same role as before.
- **The IP boxes on the right** (`172.16.0.0`, `172.16.1.0`, `172.16.2.0`) show that each subnet gets its own slice of the VPC's overall CIDR range — this is the "subnetting" from your very first slide, made concrete.

**Why two AZs matter:** if AZ1 goes down, AZ2 (with its own public + private subnet pair) keeps the application running. This is the standard AWS "highly available" architecture pattern — nearly every production VPC diagram you'll see looks like this.

_Gated-property analogy extended:_ this is like a property developer building **two identical gated communities** (AZs) in the same city (Region), connected under one HOA (VPC) — so if one community floods, residents in the other are unaffected.

|Layer|What it is|Analogy|
|---|---|---|
|**Region**|A geographic AWS location containing multiple data centers|The city|
|**Availability Zone**|An isolated data center within a region|A separate gated community in that city|
|**VPC**|Your isolated network, spanning one or more AZs, defined by a CIDR range|The HOA/property spanning multiple communities|
|**Subnet**|A slice of the VPC's IP range, tied to one AZ; public or private based on its route table|A lane within the property — street-facing or set back|
|**Route Table**|Rules deciding where a subnet's traffic goes|Signage inside the lane|
|**Internet Gateway**|Two-way door between public subnets and the internet|The property's main gate|
|**NAT Gateway**|Lets private subnets reach out to the internet, one-way only|A staff-only side gate|
|**EC2**|The actual virtual server/instance|A house on the lane|
|**Security Group**|Stateful firewall on the individual EC2 instance; allow-only, remembers return traffic|The lock on one house's front door|
|**NACL**|Stateless firewall on the entire subnet; allow or deny, evaluated in numbered order, checks both directions separately|The checkpoint at the lane's entrance|