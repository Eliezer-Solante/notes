![[Pasted image 20260716163101.png]]

### DAS(Direct Attached Storage)
![[Pasted image 20260716163358.png]]
**Definition:** Storage directly connected to a single server (via SATA, SAS, NVMe).
- **Access type:** Block-level (appears as a local disk).
- **Pros:**
    - **Low cost** and simple setup. 
    - **High performance** (no network overhead).
    - **Low latency (<0.1 ms)**.
- **Cons:**
    - Only usable by **one server** (no sharing).
    - Limited scalability (add drives to the same server). 
- **Use case:** Local workloads, standalone servers, personal PCs.

### NAS(Network Attached Storage)
![[Pasted image 20260716163552.png]] **Definition:** A dedicated file server that shares storage over a standard Ethernet network.
- **Access type:** File-level (via SMB, NFS, AFP).
- **Pros:**
    - **Centralized storage** accessible by many clients.   
    - Easy integration into existing LANs.  
    - Built-in features (snapshots, replication, permissions).  
- **Cons:**
    - **Network overhead** → higher latency (1–5 ms).  
    - Not optimal for databases or VM workloads.   
- **Use case:** Shared file storage, backups, collaboration environments.

### SAN(Storage Area Network)
![[Pasted image 20260716163730.png]]
**Definition:** A dedicated high-speed network (Fibre Channel, iSCSI) that provides block-level storage to multiple servers.
- **Access type:** Block-level (servers manage their own filesystems).
- **Pros:**
    - **High performance** and low latency (0.5–2 ms).
    - Multiple servers can access shared storage.
    - Ideal for **databases, VMs, and enterprise workloads**.
- **Cons:**
    - **High cost** and complexity.
    - Requires specialized hardware and expertise. 
- **Use case:** Enterprise data centers, mission-critical applications, large-scale virtualization.

## ⚙️ Key Characteristics

- **Block-level access** → Servers see SAN volumes as raw disks (e.g., `/dev/sdX`), and manage their own filesystems.
- **Dedicated network** → Uses Fibre Channel or iSCSI over Ethernet, separate from the LAN.
- **Scalability** → Easy to add more storage without touching servers.
- **High performance** → Low latency and high throughput, suitable for databases and virtualization.
- **Centralized management** → Storage pools can be carved into Logical Unit Numbers (LUNs) and assigned to servers.
## 📋 Typical SAN Components
- **Storage arrays** → Large disk enclosures (HDDs, SSDs).
- **SAN switches** → Specialized switches for Fibre Channel or Ethernet.
- **HBAs (Host Bus Adapters)** → Cards in servers that connect to the SAN.
- **LUNs** → Logical slices of storage presented to servers.