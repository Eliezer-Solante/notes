## 

![[Pasted image 20260715095123.png]]

---

## RPM Package Manager
- **RPM** stands for **Red Hat Package Manager**.
- It’s a **low-level package management system** used by Red Hat–based distributions such as **Fedora**, **CentOS**, and **RHEL**.
- It manages `.rpm` files — these are software bundles containing binaries, configuration files, and metadata.

### 5 modes of operation of RPM
![[Pasted image 20260715095537.png]]
## ⚙️ **Core Functions of RPM**

|Function|Description|
|---|---|
|**Install Packages**|Adds new software to your system using `.rpm` files. Example: `rpm -ivh package.rpm`|
|**Upgrade Packages**|Updates existing software to a newer version. Example: `rpm -Uvh package.rpm`|
|**Remove Packages**|Uninstalls software cleanly. Example: `rpm -e package_name`|
|**Verify Packages**|Checks integrity and authenticity using checksums and signatures. Example: `rpm -V package_name`|
|**Query Packages**|Displays information about installed packages. Example: `rpm -qa` (lists all installed packages)|

## 🔐 **Package Integrity and Security**

- RPM uses **GPG (GNU Privacy Guard)** keys to verify that packages come from trusted sources.
- This prevents tampering and ensures authenticity.
- Example:
    bash
    ```
    rpm --checksig package.rpm
    ```
    This command checks the digital signature of the package.

---
## YUM Package Manager
- YUM stands for (Yellowdog Updater, Modified)
- YUM is a **front-end tool for RPM** used in **Red Hat–based distributions** (CentOS, RHEL, Fedora before DNF).
- It automates the process of installing, updating, and removing `.rpm` packages.
- Most importantly, it **resolves dependencies automatically** — something RPM alone cannot do.

![[Pasted image 20260715100128.png]]
## 📚 **Repositories**

- YUM uses **repository configuration files** (`.repo`) stored in `/etc/yum.repos.d/`.
- Each repo file contains:
    - Base URL (where packages are stored)
    - GPG key for verification
    - Enabled/disabled status
- This allows YUM to fetch software from trusted sources automatically.
![[Pasted image 20260715100540.png]]
## 🔐 **Security**
- YUM verifies packages using **GPG signatures** before installation.
- This ensures authenticity and prevents tampered software from being installed.

### Commands
#### Installation Process
![[Pasted image 20260715100709.png]]


![[Pasted image 20260715100910.png]]

![[Pasted image 20260715101131.png]]

| Function               | Description                                  | Example Command                        |
| ---------------------- | -------------------------------------------- | -------------------------------------- |
| **Install Packages**   | Installs software and required dependencies. | `yum install httpd`                    |
| **Update Packages**    | Updates all packages or specific ones.       | `yum update`                           |
| **Remove Packages**    | Uninstalls software cleanly.                 | `yum remove httpd`                     |
| **Search Packages**    | Finds packages in repositories.yum repolist  | `yum search nginx`                     |
| **Info Packages**      | Displays details about a package.            | `yum info nginx`                       |
| **Group Install**      | Installs collections of related packages.    | `yum groupinstall "Development Tools"` |
| **History & Rollback** | Tracks transactions and allows undo.         | `yum history undo <ID>`                |
