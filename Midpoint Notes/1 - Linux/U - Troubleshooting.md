### Problem
![[Pasted image 20260715183752.png]]



#### ==First==, check hosts interface
![[Pasted image 20260715184145.png]]
no problem.

#### ==Second==, check connection between host and DNS server
![[Pasted image 20260715184122.png]]
Good.

#### ==Third==, check connectivity between the host and the repository server
![[Pasted image 20260715184426.png]]
No, connection

#### ==Fourth==, check the routing
![[Pasted image 20260715184539.png]]
no, problems on the routers.

#### ==Fifth==, check the server side and its services
![[Pasted image 20260715184752.png]]
Web server is UP, and has services

#### ==Lastly==, check the server interfaces
![[Pasted image 20260715184944.png]]
Problem detected, the server interface was DOWN

Run ![[Pasted image 20260715185038.png]]
to enable the interface

Test the URL again to Check the Website: 
![[Pasted image 20260715185206.png]]
SOLVED

