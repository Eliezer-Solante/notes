ls![[Pasted image 20260716142113.png]]

![[Pasted image 20260716142657.png]]

**The** `fdisk` **command in Linux is a menu-driven utility used to create, delete, and manage disk partitions.** It’s one of the most common tools for preparing a new hard drive or SSD before formatting and storing files.

![[Pasted image 20260716144440.png]]

![[Pasted image 20260716145422.png]]



![[Pasted image 20260716144120.png]]
It’s a concise summary of the **Linux disk setup workflow**
1. **Partition using fdisk/gdisk** → Define disk sections.
2. **Format** → Apply a filesystem type (e.g., ext4).
3. **Create mountpoint** → Make a directory for mounting.
4. **Mount filesystem** → Attach the partition to the directory.
5. **Make persistent** → Add to `/etc/fstab` so it auto-mounts on boot.
## 🧩 **1️⃣ Partition using fdisk/gdisk**
Create partitions on your disk (e.g., `/dev/sdb`).
bash
```
sudo gdisk /dev/sdb
```
Inside the interactive prompt:
- `n` → new partition
- `p` → print partition table 
- `w` → write changes

Then verify:
bash
```
sudo fdisk -l /dev/sdb
```
## 🧱 **2️⃣ Format the partition**
Apply a filesystem type such as **ext4**.
bash
```
sudo mkfs.ext4 /dev/sdb1
```
Other options:
- `mkfs.xfs /dev/sdb1` 
- `mkfs.vfat /dev/sdb1`
## 📂 **3️⃣ Create a mount point**

Make a directory where the partition will be mounted.
bash
```
sudo mkdir /mnt/data
```
## 🔗 **4️⃣ Mount the filesystem**

Attach the partition to the mount point.
bash
```
sudo mount /dev/sdb1 /mnt/data
```

Check it’s mounted:
bash
```
df -h
```
# 🔁 **5️⃣ Make the mount persistent**
Add an entry to `/etc/fstab` so it auto-mounts at boot.
1. Get the UUID: 
    bash 
    ```
    sudo blkid /dev/sdb1
    ```
2. Edit `/etc/fstab`:
    bash
    ```
    sudo nano /etc/fstab
    ```
    Add a line like:
    Code
    ```
    UUID=xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx /mnt/data ext4 defaults 0 2
    ```
3. Test:
    bash
    ```
    sudo mount /dev/sdb1
    ```
       or 
    ```
sudo sudo umount /dev/sdb1
    ```

![[Pasted image 20260716145827.png]]
![[Pasted image 20260716150108.png]]
![[Pasted image 20260716150231.png]]



