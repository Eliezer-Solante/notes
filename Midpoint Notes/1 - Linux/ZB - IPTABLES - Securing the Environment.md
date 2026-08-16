
## CHAIN INPUT RULE
Input rule on the App Server that will allow SSH from the client/laptop
![[Pasted image 20260716093606.png]]
In this case, Client B can access the server via SSH because of the default rule. The goal is that only the registered client can access the server and it has no rule yet.

DROP Rule
![[Pasted image 20260716094139.png]]
![[Pasted image 20260716094347.png]]


## CHAIN OUTPUT RULE
![[Pasted image 20260716094525.png]]

![[Pasted image 20260716094647.png]]

### Rule Insertion
![[Pasted image 20260716094924.png]]
![[Pasted image 20260716094945.png]]

### Rule Delete
![[Pasted image 20260716095032.png]]

Making sure that the database server is only accepting traffic from the application server
NOTE: Configure it on the database server side.
![[Pasted image 20260716095153.png]]


Summary
![[Pasted image 20260716095437.png]]

![[Pasted image 20260716100138.png]]

