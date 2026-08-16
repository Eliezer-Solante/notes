
![[Pasted image 20260714192406.png]]
## Main File Types in Linux

Linux treats **everything as a file**, but not all files are the same. Here are the major categories:

- **Regular files**
    - Contain user data: text, programs, images, documents.
    - Most common type.
    - Example: `/home/eliezer/report.txt`
- **Directory files**
    - Special files that act as containers for other files.
    - Example: `/etc`, `/home`
- **Character device files**
    - Represent devices that handle data character by character.
    - Example: `/dev/tty` (terminal), `/dev/random`
- **Block device files**
    - Represent devices that handle data in blocks (like disks).
    - Example: `/dev/sda`, `/dev/nvme0n1`
- **Socket files**
    - Enable communication between processes (IPC).
    - Example: `/var/run/docker.sock`
- **FIFO (named pipes)**
    - Allow one process to pass data to another in sequence.
    - Example: `/tmp/myfifo`
- **Links**
    - **Hard links**
        - Direct references to the same inode (the actual data on disk).
        - Indistinguishable from the original file — deleting one does not remove the data until all links are gone.
        - Example:
            bash
        ```
        ln original.txt hardlink.txt
        ```
        Both names point to the same file content.
    - **Symbolic links (Soft links)**
        - Pointers to another file’s path (like a shortcut).
        - If the target file is deleted, the symlink becomes broken.
        - Example: `/usr/bin/python → /usr/bin/python3.10`
            bash

```
ln -s original.txt softlink.txt
```


## 📌 How to Identify File Types

You can use commands like:
bash
```
ls -l
```

Output example:
```
-rw-r--r--  1 user user  1234 Jul 14  report.txt   # Regular file
drwxr-xr-x  2 user user  4096 Jul 14  Documents    # Directory
lrwxrwxrwx  1 root root    10 Jul 14  python -> python3.10  # Symbolic link
```

- The **first character** in the permissions string tells you the type:
    - `-` → Regular file       
    - `d` → Directory     
    - `c` → Character device      
    - `b` → Block device       
    - `l` → Symbolic link       
    - `p` → FIFO (pipe)      
    - `s` → Socket
        

## 🎓 College-Level Analogy

Think of Linux file types like **different kinds of doors in a building**
- Regular files → Rooms with content (books, furniture).
- Directories → Hallways that connect rooms.
- Device files → Special doors that lead to machines (printers, disks)
- Sockets/FIFOs → Communication tubes between rooms.
- Symbolic links → Signposts pointing to another room.
    
## 📊 Summary Table

|File Type|Symbol|Example|Purpose|
|---|---|---|---|
|**Regular**|`-`|`report.txt`|Store data|
|**Directory**|`d`|`/etc`|Organize files|
|**Character device**|`c`|`/dev/tty`|Stream data char by char|
|**Block device**|`b`|`/dev/sda`|Access disks in blocks|
|**Socket**|`s`|`/var/run/docker.sock`|Process communication|
|**FIFO**|`p`|`/tmp/myfifo`|Sequential process communication|
|**Symbolic link**|`l`|`python → python3.10`|Shortcut to another file|
![[Pasted image 20260714192510.png]]