![[Pasted image 20260715111131.png]]
### DPKG
- **DPKG** stands for **Debian Package**.
- It’s a **low-level package manager** that works directly with `.deb` files.
- It installs, removes, and queries packages, but **does not handle dependencies automatically**.
![[Pasted image 20260715111450.png]]

## **Core Functions of DPKG**

| Function             | Description                         | Example Command                    |
| -------------------- | ----------------------------------- | ---------------------------------- |
| **Install Packages** | Installs a `.deb` file directly.    | `dpkg -i package.deb`              |
| **Remove Packages**  | Uninstalls a package.               | `dpkg -r package_name`             |
| **List Packages**    | Shows installed packages.           | `dpkg -l`                          |
| **Query Packages**   | Displays info about a package.      | `dpkg -s package_name`             |
| **Unpack Packages**  | Extracts files without configuring. | `dpkg -x package.deb /path/to/dir` |


---

### APT(Advanced Package Tool)
- APT is a **high-level package manager** built on top of **DPKG**.
- It automates installation, removal, updates, and dependency resolution for `.deb` packages.
- It pulls software from **repositories** defined in configuration files.

![[Pasted image 20260715112003.png]]

![[Pasted image 20260715112019.png]]
## ⚙️ **APT Command Overview**

|Function|Command|Purpose|
|---|---|---|
|**Update Package Lists**|`apt update`|Refreshes the list of available packages from repositories.|
|**Upgrade Packages**|`apt upgrade`|Upgrades all installed packages to the latest versions.|
|**Install Software**|`apt install vim`|Installs a package and its dependencies.|
|**Remove Software**|`apt remove vim`|Uninstalls a package but keeps configuration files.|
|**Search Packages**|`apt search nginx`|Searches repositories for a package.|
|**Show Package Info**|`apt show nginx`|Displays details about a package.|
|**Edit Sources**|`apt edit-sources`|Opens `/etc/apt/sources.list` in your default editor to add/remove repositories.|
|**List Packages**|`apt list`|Displays available and installed packages (can be narrowed with options like `--installed` or `--upgradable`).|
