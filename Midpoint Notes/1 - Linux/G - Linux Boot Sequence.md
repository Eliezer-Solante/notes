
![[Pasted image 20260714185354.png]]
## 1️⃣ **BIOS POST (Power-On Self Test)**

- When you press the power button, the **BIOS/UEFI firmware** runs first.
- It performs a **POST** (Power-On Self Test) to check basic hardware: CPU, RAM, keyboard, storage devices.
- If something critical fails (like RAM missing), the system won’t continue.
- Once checks pass, BIOS looks for a **bootable device** (hard drive, SSD, USB, etc.) and hands control to the **boot loader**.

👉 Think of BIOS POST as the “hardware referee” making sure the playing field is ready.

## 2️⃣ **Boot Loader (GRUB2)**

- The **boot loader** is a small program stored on the bootable device (usually in the MBR or EFI partition).
- In Linux, the most common boot loader is **GRUB2**.
- GRUB2’s job:
    - Present a menu (if multiple OS kernels are available).
    - Load the selected **Linux kernel** into memory.
    - Pass control to the kernel.
- It can also pass parameters (like `quiet` or `single-user mode`) to influence how the kernel starts.

👉 GRUB2 is like the “dispatcher” — it decides which operating system or kernel to launch.

## 3️⃣ **Kernel Initialization**

- The **Linux kernel** is now in charge.
- It initializes:
    - CPU, memory management, and process scheduling.  
    - Device drivers for storage, networking, USB, etc.     
    - Mounts the **root filesystem** (so the OS can access files).        
- Once hardware and drivers are ready, the kernel starts the **init process** (traditionally `init`, now usually `systemd`).
    
👉 The kernel is the “engine” — it powers up all hardware and prepares the environment for user-space programs.

## 4️⃣ **Init Process (systemd)**

- **systemd** is the modern init system in most Linux distros.
- It’s the first user-space process (PID 1).
- Responsibilities:
    - Start essential services (networking, logging, graphical interface).
    - Launch background daemons.
    - Manage dependencies between services.
    - Finally, bring up the **login prompt** or graphical desktop.
- systemd also keeps running to supervise services, restart them if they fail, and handle shutdown/reboot.

👉 systemd is the “orchestra conductor” — it ensures all services start in the right order and keeps them running smoothly.