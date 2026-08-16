![[Pasted image 20260716170235.png]]
## 🧩 What LVM Is
- **LVM** is a system for managing disk storage that lets you create, resize, and move logical volumes more easily than fixed partitions.
- Instead of binding filesystems directly to physical partitions, LVM adds a layer of abstraction:
    - **Physical Volumes (PV)** → actual disks or partitions.
    - **Volume Groups (VG)** → pools of storage made from one or more PVs.
    - **Logical Volumes (LV)** → slices of a VG that act like partitions, where you put filesystems.

![[Pasted image 20260716170458.png]]
![[Pasted image 20260716170537.png]]

![[Pasted image 20260716170721.png]]

![[Pasted image 20260716171156.png]]
NOTE: before running `mount`command, check if mounting point/directory (`/mnt/vol1`) exists, then create using `mkdir /mnt/vol1`

Resizing Volume
![[Pasted image 20260716172858.png]]

![[Pasted image 20260716173030.png]]


## ⚙️ Workflow
1. <mark style="background: #FFB8EBA6;">**Partition disk**</mark> → Create a partition for LVM use.
2. <mark style="background: #FFB8EBA6;">**Create PV**</mark> → Initialize the partition for LVM.
    bash
    ```
    sudo pvcreate /dev/sdb1
    ```
3. <mark style="background: #FFB8EBA6;">**Create VG**</mark> → Combine PVs into a storage pool.
    bash
    ```
    sudo vgcreate my_vg /dev/sdb1
    ```
4. **<mark style="background: #FFB8EBA6;">Create LV</mark>** → Allocate space from the VG
    bash
    ```
    sudo lvcreate -L 5G -n my_lv my_vg
    ```
5. **<mark style="background: #FFB8EBA6;">Format & mount</mark>** → Treat LV like a partition.
    bash
    ```
    sudo mkfs.ext4 /dev/my_vg/my_lv
    sudo mkdir /mnt/data --if no mounting point exist--
    sudo mount /dev/my_vg/my_lv /mnt/data
    ```
6. <mark style="background: #FFB8EBA6;">Make the mount persistent</mark>
    - <mark style="background: #FFF3A3A6;">Edit fstab with vim</mark>
        - Add a new line to make the mount persistent.
            - Run `sudo vim /etc/fstab
        - Go to the last line in vim (`G`)
        - Press `o` to open a new line
        - Add: `/dev/vg/lv /mnt/vol1 ext4 rw 0 0`
        - Save and exit with `:wq`
    
    - <mark style="background: #FFF3A3A6;">Alternative: echo | tee</mark>
        - You can append the entry without opening vim.
            - Run `echo '/dev/vg/lv /mnt/vol1 ext4 rw 0 0' | sudo tee -a /etc/fstab`
        - This appends the line directly to fstab
        - Useful for scripting or automation
        
7. <mark style="background: #FFB8EBA6;">Test the configuration</mark>
    - Verify that the entry works before rebooting.
        - Run `sudo mount -a`
    - If no errors appear, the mount is successful
    - Check with `df -h` to confirm the LV is mounted


![[Pasted image 20260716144137.png]]
This line outlines the **Logical Volume Management (LVM)** setup process in Linux:
1. **Partition using fdisk/gdisk** → Prepare the disk by creating partitions.
2. **Create PV (Physical Volume)** → Initialize the partition for LVM use.    
3. **Create VG (Volume Group)** → Combine one or more physical volumes into a group.    
4. **Create LV (Logical Volume)** → Allocate space from the volume group for actual use (mounting, formatting, etc.).