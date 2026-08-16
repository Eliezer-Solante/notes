![[Pasted image 20260805181728.png]]
![[Pasted image 20260806091511.png]]
This image lays out the core building blocks of a **Virtual Private Cloud (VPC)** — the four pillars you need to understand to design a secure, functional cloud network. Let's go through each one.

## What a VPC actually is

A VPC is your own logically isolated slice of a cloud provider's network (AWS, Azure, GCP, etc.). Think of it as your private data center carved out of shared physical infrastructure — you control the IP ranges, subnets, routing, and security, even though the underlying hardware is shared with other customers.

## The four pillars shown

**1. Subnetting (IP Address)** You divide your VPC's IP address range (a CIDR block, like `10.0.0.0/16`) into smaller subnets. Typically:

- **Public subnets** — host resources that need direct internet access (load balancers, bastion hosts)
- **Private subnets** — host resources that should stay internal (databases, app servers)

This segmentation is the foundation everything else builds on.

**2. Routing (Route Tables)** Route tables decide where traffic goes. Each subnet is associated with a route table that says things like "traffic to the internet goes through this gateway" or "traffic to another VPC goes through this peering connection." Without correct routes, subnets are isolated even if they're technically networked.

**3. Firewalls (NACLs and Security Groups)** Two layers of traffic filtering:

- **Security Groups** — stateful, attached to individual resources (like an instance), control inbound/outbound traffic at that level
- **NACLs (Network Access Control Lists)** — stateless, attached to subnets, act as a broader perimeter filter

Using both gives you defense in depth — one mistake in a security group doesn't necessarily expose the whole subnet.

**4. Gateways** These are the doors in and out of your VPC:

- **Internet Gateway** — lets public subnet resources reach/be reached by the internet
- **NAT Gateway** — lets private subnet resources initiate outbound internet traffic (like pulling updates) without being directly reachable from outside
- **VPN/Direct Connect Gateway** — connects your VPC back to on-premises infrastructure

## Why this sequencing matters

The diagram's order isn't arbitrary — it mirrors how you'd actually design a VPC:

1. First carve up your IP space (subnetting)
2. Then decide how traffic flows between pieces (routing)
3. Then lock down what's allowed to talk to what (firewalls)
4. Then decide how the whole thing connects to the outside world (gateways)

![[Pasted image 20260806091543.png]]
![[Pasted image 20260806091533.png]]
![[Pasted image 20260806091724.png]]


