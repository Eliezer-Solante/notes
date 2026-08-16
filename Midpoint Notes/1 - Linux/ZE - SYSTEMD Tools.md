### SYSTEMCTL
![[Pasted image 20260716115230.png]]
**is the command-line tool used to control and manage the** `systemd` **system and service manager on modern Linux distributions.** It lets you start, stop, enable, disable, reload, and check the status of services and system units.

#### Commands
![[Pasted image 20260716115753.png]]

![[Pasted image 20260716115803.png]]

![[Pasted image 20260716115815.png]]

- ==Check what port the service is using:==
```
sudo ss -tulpn | grep <service_name>

```

![[Pasted image 20260716120203.png]]
### 🌀 `systemctl daemon-reload`
- **Purpose:** Reloads the systemd manager configuration.
- **When to use:** After you **edit or create** a unit file (like a `.service` file) manually.
- **Effect:** Tells systemd to re-read all unit files so it recognizes your changes.
- **Example scenario:** You modify `/etc/systemd/system/project-mercury.service`. Before restarting the service, you run:
    bash
    ```
    sudo systemctl daemon-reload
    ```
    This ensures systemd applies the updated configuration.
### 🧩 `systemctl edit project-mercury.service --full`
- **Purpose:** Opens the **full service file** for editing in your default text editor. 
- `--full` **flag:** Loads the entire unit file (not just an override snippet).
- **Effect:** Lets you directly modify the service’s configuration, such as `ExecStart`, `Restart`, or `Environment` directives.  
- **Example scenario:** You want to change how the `project-mercury` service starts. You run: 
    bash
    ```
    sudo systemctl edit project-mercury.service --full
    ```
    After saving, you’d typically run `systemctl daemon-reload` and then `systemctl restart project-mercury.service` to apply changes.
### ⚙️ Typical Workflow
1. Edit the service file → `systemctl edit project-mercury.service --full`  
2. Reload systemd configs → `systemctl daemon-reload`
3. Restart the service → `systemctl restart project-mercury.service`

Runlevels
![[Pasted image 20260716125342.png]]

Display all units whether active or inactive
![[Pasted image 20260716125243.png]]

if you want to display just the active unit
![[Pasted image 20260716125224.png]]

---

### JOURNALCTL
![[Pasted image 20260716115253.png]]
Controls and queries the systemd journal
- **View logs**: Displays logs from the systemd journal.
- **Filter logs**: Narrow down logs by service, time, priority, or boot session.
- **Persistent logging**: If configured, logs are stored across reboots.
- **Troubleshooting**: Essential for diagnosing service failures and system issues.

![[Pasted image 20260716125524.png]]

### 🧾 `journalctl`
- **Purpose:** Shows **all logs** collected by the systemd journal.
- **Use case:** General log inspection — kernel, services, and system messages.
- **Example:**
    bash
    ```
    journalctl
    ```
    ➡ Displays everything from oldest to newest.
### 🔄 `journalctl -b`
- **Purpose:** Shows logs **from the current boot** only.
- **Use case:** Useful for troubleshooting startup or runtime issues since the last reboot.
- **Example:**
    bash
    ```
    journalctl -b
    ```
    ➡ Filters logs to the current boot session
### ⚙️ `journalctl -u UNIT`
- **Purpose:** Displays logs for a **specific systemd unit** (service).
- **Use case:** Focused debugging of one service, e.g., `nginx.service` or `project-mercury.service`. 
- **Example:**
    bash
    ```
    journalctl -u project-mercury.service
    ```
    ➡ Shows only logs related to that service.
    ![[Pasted image 20260716125706.png]]
    

