![[Pasted image 20260805094208.png]]
![[Pasted image 20260805094612.png]]


![[Pasted image 20260805094555.png]]
![[Pasted image 20260805094632.png]]
**1. Design for failure**  
Assume that any component — a server, disk, availability zone, or even an entire region — can and will fail at some point. Instead of trying to prevent failure entirely, build systems that detect failures automatically and recover gracefully, with no single point of failure. Practically, this means using multiple Availability Zones, auto-scaling groups that replace unhealthy instances, health checks, and redundancy at every layer.

![[Pasted image 20260805094708.png]]
![[Pasted image 20260805094759.png]]
**2. Decouple components**  
Avoid tightly binding application components together so that a failure or slowdown in one doesn't cascade and take down the whole system. Instead, use intermediaries like message queues (SQS), event buses (EventBridge), or load balancers so components can operate, fail, scale, and be updated independently. This also makes it easier to swap out or upgrade individual pieces without touching the rest of the system.

![[Pasted image 20260805094923.png]]
![[Pasted image 20260805095232.png]]
**3. Implement elasticity**  
Design your infrastructure to automatically scale resources up or down based on actual demand, rather than provisioning for peak capacity all the time. This means using services like Auto Scaling Groups, Lambda, or Fargate so you pay only for what you need at any given moment — improving both cost-efficiency and resilience to traffic spikes.

![[Pasted image 20260805095405.png]]
![[Pasted image 20260805095423.png]]
**4. Think parallel**  
Rather than relying on a single large resource to do all the work sequentially, break tasks into smaller units that can run concurrently across multiple resources. This improves performance, throughput, and fault tolerance — for example, distributing a batch job across many Lambda functions or EC2 instances instead of running it on one large machine.


![[Pasted image 20260805095502.png]]
![[Pasted image 20260805095515.png]]