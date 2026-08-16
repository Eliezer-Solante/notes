![[Pasted image 20260716164207.png]]

- **NFS** is a **distributed file system protocol** developed by Sun Microsystems.
- It allows a system to **share directories and files** with others over a network.
- Clients can mount remote directories via NFS and interact with them **as if they were local files**.
## ⚙️ How NFS Works
1. **Server setup** → A machine exports directories using NFS.
2. **Client mount** → Other machines mount those directories over the network.
3. **Transparent access** → Files are read/written as if they were on the local disk.

Protocols used:
- **RPC** (Remote Procedure Call) for communication.
- Runs over **TCP/UDP**, typically on port **2049**.

## 📋 Key Features
- **File-level sharing** → Unlike SAN (block-level), NFS shares files
- **Cross-platform** → Works across Linux, UNIX, BSD, and even Windows.
- **Centralized storage** → One server can host files for many clients.
- **Transparency** → Clients don’t need to know files are remote.

![[Pasted image 20260716164718.png]]
![[Pasted image 20260716164809.png]]