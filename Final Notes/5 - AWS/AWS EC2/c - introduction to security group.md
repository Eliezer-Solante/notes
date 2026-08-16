**Security Group**  
A virtual firewall attached to individual resources (like an EC2 instance) that controls inbound/outbound traffic at that resource level. It's stateful — if you allow traffic in, the response traffic out is automatically allowed too.  
_Analogy: the lock and guest list on an individual house's front door — even if you're inside the gated property and walked past the main gate, you still can't get into a specific house unless you're on that house's approved visitor list. And if the homeowner lets you in, you're automatically allowed back out._

To place it in your list, it fits right after NAT Gateway, since it's the next layer down — from "which gate can traffic use" to "which specific house can it enter":

**How they connect (updated):** VPC → divided into subnets → subnets use route tables → route tables point public subnets to the IGW (two-way) or private subnets to the NAT Gateway (outbound-only) → individual resources within a subnet are further protected by security groups, which decide exactly who's allowed at their front door.

If you also want **NACLs** (the subnet-level, stateless counterpart to security groups) worked in for contrast, the analogy would be: _the checkpoint at the entrance to each lane, checking everyone who enters or exits — unlike the house's own lock, this guard doesn't remember you, so it checks both your way in and your way out separately._ Want me to add that one too, or turn the whole set into a labeled diagram like before?



![[Pasted image 20260806133612.png]]

This shows how the security group evaluates traffic against its rule tables:

- Client **A** comes in from anywhere (`0.0.0.0/0`) — matches the first inbound rule, so it's allowed through on port 80.
- Client **B** comes from a specific IP (`121.10.13.141`) — matches the second, more restrictive rule, also allowed on port 80.
- Traffic that doesn't match any inbound rule gets dropped silently (no explicit "deny" needed — security groups are default-deny).
- The outbound rule here is wide open (`All / All / 0.0.0.0/0`) — the default AWS gives new security groups, letting the EC2 instance respond to anything and reach out freely.

Since security groups are **stateful**, that outbound rule usually doesn't even need to explicitly allow the return traffic to clients A and B — once a request is let in, the response is automatically permitted back out, regardless of what the outbound table says.