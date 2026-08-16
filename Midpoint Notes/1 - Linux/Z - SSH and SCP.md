
## SSH(Secure Shell) 
- port 22

![[Pasted image 20260715221644.png]]
Connecting to a remote server 
- remote server must have a running SSH service to be accessed remotely
<mark style="background: #FF5582A6;">NOTE: In Accessing via SSH, do not use `sudo` command as it will enter as the root user and not the client</mark> 

![[Pasted image 20260715221856.png]]

![[Pasted image 20260715222117.png]]
Generating Key Pair

![[Pasted image 20260715222252.png]]
Copying the public key to the remote server `bob@devapp01`

![[Pasted image 20260715222350.png]]
Location of the public key on the remote server


---
## SCP(Secure Copy Protocol)
- It is a command-line utility used to securely transfer files and directories between a local host and a remote host, or between two remote hosts.
- allows you to copy data over SSH

![[Pasted image 20260715223251.png]]




