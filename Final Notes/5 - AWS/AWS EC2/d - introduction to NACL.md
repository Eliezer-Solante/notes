**NACL (Network Access Control List)**  
A stateless firewall attached at the **subnet** level, not the individual resource. Because it's stateless, it must explicitly allow both the inbound request _and_ the outbound response — nothing is remembered between them. Rules are evaluated in numbered order (lowest number first), and each rule can be **Allow** or **Deny**, unlike security groups which are allow-only.  
_Analogy: the checkpoint at the entrance to the lane itself — every car is checked coming in and checked again going out, and the guard doesn't remember your last visit._

![[Pasted image 20260806134234.png]]

![[Pasted image 20260806134241.png]]
A few things worth noticing here:

- Rule **100** is checked before rule **200** — the moment a request matches, evaluation stops. So a request from `0.0.0.0/0` on port 80 hits rule 100 and gets allowed, even though rule 200 exists further down.
- The **`*` rule** at the bottom is the implicit "deny everything else" that every NACL has by default — you can't remove it, only make earlier rules more permissive.
- Notice the outbound rule allows ports **1024-65535** rather than mirroring the inbound port 80. That's because NACLs are stateless — the _response_ to an inbound web request comes back on a random high-numbered "ephemeral" port, so you have to explicitly allow that range outbound, or return traffic gets silently dropped. This is the exact opposite of a security group, where you don't need a matching outbound rule at all.