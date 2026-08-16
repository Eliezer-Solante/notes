#### **Package manager** 
- is a specialized tool in Linux (and other operating systems) that automates the process of installing, updating, configuring, and removing software. Instead of manually downloading programs and figuring out where to put files, the package manager does all the heavy lifting for you.

#### Functions of Package Manager
### 🧩 **Package Integrity and Authenticity**

- Ensures that the software you install hasn’t been tampered with.
- Package managers verify **digital signatures** and **checksums** to confirm authenticity.
- Example: When you run `yum install`, it checks the GPG key of the repository to ensure the package is genuine and safe.
### ⚙️ **Simplified Package Management**

- Instead of manually downloading and configuring software, package managers automate everything.
- They handle installation, updates, and removal with simple commands like `yum install nginx`.
- This reduces human error and saves time, especially on servers or large systems.    
### 📦 **Grouping Packages**

- Packages can be grouped into **collections** or **modules** for easier management.
- Example: Installing the “Development Tools” group in CentOS automatically installs compilers, debuggers, and libraries needed for coding. 
    bash
    ```
    yum groupinstall "Development Tools"
    ```
- This helps maintain consistency across systems.
### 🔗 **Manage Dependencies**

- One of the most powerful features: automatically installs required libraries or tools.
- If you install a web server that needs SSL libraries, YUM fetches them automatically.
- Prevents “dependency hell,” where missing components break software.

#### Types of Package Managers
![[Pasted image 20260715094410.png]]

## 🔴 **Low-Level Package Managers**

These work directly with package files and metadata.
### **DPKG**
- Used in **Debian-based systems** (like Ubuntu). 
- Handles `.deb` packages directly.   
- Commands:
    bash
    
    ```
    dpkg -i package.deb   # install
    dpkg -r package_name  # remove
    dpkg -l               # list installed packages
    ```
- Doesn’t automatically resolve dependencies — you must install them manually or use APT. 
### **RPM**
- Used in **Red Hat-based systems** (like Fedora, CentOS, RHEL).  
- Handles `.rpm` packages. 
- Commands:
    bash
    ```
    rpm -ivh package.rpm   # install
    rpm -e package_name    # remove
    rpm -qa                # list installed packages
    ```
- Also doesn’t handle dependencies automatically — that’s YUM or DNF’s job.

## 🟢 **High-Level Package Managers**

These sit on top of DPKG or RPM and manage dependencies, repositories, and updates.
### **APT**
- Stands for **Advanced Package Tool**. 
- Works with DPKG to install `.deb` packages and resolve dependencies automatically.
- Commands:
    bash
    ```
    apt install vim
    apt update
    apt upgrade
    ```
- Simplifies system maintenance and ensures consistency.  
### **YUM**
- Stands for **Yellowdog Updater, Modified**.
- Works with RPM to install `.rpm` packages and handle dependencies.
- Commands:
    bash
    ```
    yum install nginx
    yum update
    yum remove nginx
    ```
- Uses `.repo` files to locate repositories.
    
## 🟡 **Modern or Legacy Interfaces**

These are either older command-line tools or newer replacements.
### **APT-GET**
- Older command-line interface for APT.   
- Still widely used in scripts and automation.  
- Example:  
    bash
    ```
    apt-get install curl
    apt-get update
    ```
- APT is now preferred for interactive use because it provides better progress and error messages.
### **DNF**
- Stands for **Dandified YUM** — the modern replacement for YUM.
- Faster, more reliable, and supports rollback of transactions. 
- Commands:
    bash
    ```
    dnf install httpd
    dnf update
    dnf history undo <ID>
    ```
- Used in Fedora, RHEL 8+, and CentOS Stream.

|**Tool**|**Base System**|**Package Type**|**Dependency Handling**|**Modern Status**|
|---|---|---|---|---|
|**DPKG**|Debian/Ubuntu|`.deb`|Manual|Still used (low-level)|
|**RPM**|Red Hat/Fedora|`.rpm`|Manual|Still used (low-level)|
|**APT**|Debian/Ubuntu|`.deb`|Automatic|Active|
|**YUM**|Red Hat/CentOS|`.rpm`|Automatic|Legacy|
|**APT-GET**|Debian/Ubuntu|`.deb`|Automatic|Legacy|
|**DNF**|Fedora/RHEL|`.rpm`|Automatic|Modern replacement for YUM|