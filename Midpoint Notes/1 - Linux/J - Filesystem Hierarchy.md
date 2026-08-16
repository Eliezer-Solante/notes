![[Pasted image 20260714193818.png]]

The **Filesystem Hierarchy Standard (FHS)** defines a consistent structure across Linux systems. Everything begins at the **root directory (**`/`**)**, and from there, specialized subdirectories branch out.

## 📂 **Key Directories**

| Directory    | Purpose                                                          | Example Contents                     |
| ------------ | ---------------------------------------------------------------- | ------------------------------------ |
| **/ (root)** | The top-level directory; all other directories branch from here. | `/bin`, `/home`, `/usr`              |
| **/bin**     | Essential user commands needed for booting and repair.           | `ls`, `cp`, `mv`, `cat`              |
| **/boot**    | Contains bootloader files and the Linux kernel.                  | `vmlinuz`, `initrd.img`, `grub.cfg`  |
| **/dev**     | Device files representing hardware (disks, terminals, etc.).     | `/dev/sda`, `/dev/tty`               |
| **/etc**     | System-wide configuration files.                                 | `passwd`, `fstab`, `hostname`        |
| **/home**    | Personal directories for each user.                              | `/home/eliezer`, `/home/student`     |
| **/lib**     | Shared libraries needed by system programs.                      | `libc.so`, `modules/`                |
| **/media**   | Mount points for removable media.                                | `/media/usb`, `/media/cdrom`         |
| **/mnt**     | Temporary mount point for filesystems.                           | `/mnt/backup`, `/mnt/testdrive`      |
| **/opt**     | Optional or third-party software packages.                       | `/opt/google`, `/opt/vscode`         |
| **/tmp**     | Temporary files deleted on reboot.                               | Session data, installer caches       |
| **/usr**     | User programs and utilities (non-essential for boot).            | `/usr/bin`, `/usr/lib`, `/usr/share` |
| **/var**     | Variable data that changes over time.                            | Logs, mail, spool files              |
![[Pasted image 20260714194005.png]]
## 📊 Hierarchy Visualization

Code

```
/
├── bin
├── boot
├── dev
├── etc
├── home
├── lib
├── media
├── mnt
├── opt
├── tmp
├── usr
└── var
```

👉 Notice: all of these are **direct children of** `/`, not ranked or ordered by importance. They are organized by function.

Think of Linux as a **university campus**:

- `/` → The main gate.
- `/bin` → Classrooms with essential tools.
- `/etc` → Administrative office storing policies. 
- `/home` → Dorm rooms for each student (user). 
- `/usr` → Library and labs with shared resources.    
- `/var` → Bulletin boards and mailrooms that constantly update.   
- `/tmp` → Temporary lockers cleared daily.