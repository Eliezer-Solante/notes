![[Pasted image 20260714190911.png]]

![[Pasted image 20260714190841.png]]
![[Pasted image 20260714191034.png]]
.

## 🔹 Runlevel 3 (Multi-user, Text Mode)

- **SysV init meaning**: Multi-user mode with networking enabled, but **no graphical interface**.
- **systemd equivalent**: `multi-user.target`.   
- **What you get**
    - Multiple users can log in simultaneously (via console or SSH).
    - Networking services are active (so servers can run).  
    - You interact with the system through a **command-line interface (CLI)** only.  
- **Use cases**:
    - Servers (web servers, database servers, etc.) where a GUI is unnecessary. 
    - Troubleshooting graphics issues (booting without GUI).   
    - Lightweight environments where resources are conserved.
    
👉 Think of runlevel 3 as **“server mode”** — full functionality, but text-only.

## 🔹 Runlevel 5 (Multi-user, Graphical Mode)

- **SysV init meaning**: Multi-user mode with networking **and graphical interface**.
- **systemd equivalent**: `graphical.target`.   
- **What you get**: 
    - Everything from runlevel 3 (multi-user + networking).      
    - A **graphical login manager** (like GDM, LightDM, or SDDM).     
    - Desktop environments (GNOME, KDE, XFCE, etc.) start automatically.    
- **Use cases**: 
    - Desktop systems where users expect a GUI.    
    - Workstations and laptops.     
    - Any scenario where graphical applications are needed.
        
👉 Think of runlevel 5 as **“desktop mode”** — same as runlevel 3, but with a GUI layered on top.

| Runlevel | Meaning                                     | Typical Use Case                     |     |
| -------- | ------------------------------------------- | ------------------------------------ | --- |
| **0**    | Halt                                        | Shuts down the system                |     |
| **1**    | Single-user mode                            | Maintenance, recovery, no networking |     |
| **2**    | Multi-user (no networking, distro-specific) | Rarely used                          |     |
| **3**    | Multi-user with networking (text mode)      | Servers, headless systems            |     |
| **4**    | Undefined/custom                            | Admin-defined special mode           |     |
| **5**    | Multi-user with GUI                         | Desktop systems                      |     |
| **6**    | Reboot                                      | Restarts the system                  |     |
