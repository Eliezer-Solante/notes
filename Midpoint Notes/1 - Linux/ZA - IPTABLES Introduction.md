![[Pasted image 20260716091033.png]]

### IPTABLES

**![[Pasted image 20260716091552.png]]

In Ubuntu, IPTABLES should be installed manually in the server side
![[Pasted image 20260716092012.png]]

Chain INPUT 
- Incoming Traffic
- SSH Connection
Chain OUTPUT
- Actions initiated by this server to other systems
- ex. server initiates connection to a database server to query or write data
Chain FORWARD
- Typically used in network routers, where data is forwarded to other devices on the network
- Not Commonly Used

NOTE: If the tables are empty, the default policy is to accept all traffic IN and OUT

Chain Rules
![[Pasted image 20260716093026.png]]
