![[Pasted image 20260806125501.png]]
**VPC (Virtual Private Cloud)** Your own isolated network inside AWS, defined by a CIDR IP range (e.g., `10.0.0.0/16`). Everything else below lives inside it. 
_Analogy: a lane or facility inside a gated property — e.g., "Housing Lane," isolated from the other lanes even though they're all on the same land._

**Subnet** A subdivision of the VPC's IP range, tied to a specific Availability Zone. Subnets are labeled public (has a route to the internet) or private (no direct internet route) based on their route table. _Analogy: a section within the lane — houses facing the street (public) vs. houses set back behind a fence (private)._

**Route Table** A set of rules that decides where network traffic goes. Each subnet is linked to one route table. The key rule that makes a subnet "public" is a route sending `0.0.0.0/0` traffic to an Internet Gateway. 
_Analogy: the signage inside the lane — "this path leads to the main gate" vs. "this path only leads to the courtyard."_

**Internet Gateway (IGW)** Attached to the VPC, it's the door that allows two-way internet traffic for resources in public subnets (e.g., a web server that needs to be reachable from outside). 
_Analogy: the main gate of the property — the one official entrance/exit to the public road._

**NAT Gateway** Sits in a public subnet but serves private subnets — it lets private resources (like a database) reach out to the internet (for updates, patches) without allowing anything in from outside. 
_Analogy: a staff-only side gate — residents in the back houses can walk out to run errands, but nobody from the street can walk in through it._

**How they connect:** VPC → divided into subnets → subnets use route tables → route tables point public subnets to the IGW (two-way) or private subnets to the NAT Gateway (outbound-only). _Analogy: the gated property is divided into lanes, each lane has its own signage, and the signage routes residents either to the main gate or the staff side gate._