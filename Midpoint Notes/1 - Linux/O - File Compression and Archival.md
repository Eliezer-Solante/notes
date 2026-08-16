cd #### Viewing File Sizes
![[Pasted image 20260715130944.png]]

---

#### Archiving Files

![[Pasted image 20260715131338.png]]

- **Archiving** means bundling multiple files or directories into a single file.
- It does **not** reduce file size — it’s about organization and portability.
- The most common tool is **tar** (short for _tape archive_).

## ⚙️ **Key tar Commands**

|Function|Command|Purpose|
|---|---|---|
|**Create Archive**|`tar -cvf archive.tar /path/to/files`|Collects files into one `.tar` archive.|
|**Extract Archive**|`tar -xvf archive.tar`|Unpacks files from the archive.|
|**List Contents**|`tar -tvf archive.tar`|Shows what’s inside without extracting.|
|**Append Files**|`tar -rvf archive.tar newfile.txt`|Adds files to an existing archive.|
|**Update Archive**|`tar -uvf archive.tar file.txt`|Updates files if they’ve changed.|

Additional

|Goal|Command Structure|
|---|---|
|**Create a standard tar archive**|`tar -cvf archive.tar /path/to/folder`|
|**Create a compressed gzip archive**|`tar -czvf archive.tar.gz /path/to/folder`|
|**Extract a regular tar archive**|`tar -xvf archive.tar`|
|**Extract a gzip archive**|`tar -xzvf archive.tar.gz`|
|**Extract to a specific folder**|`tar -xvf archive.tar -C /target/directory`|
|**List archive contents**|`tar -tvf archive.tar`|
---

#### Compression
![[Pasted image 20260715132001.png]]
- **Compression** reduces the size of files using algorithms.
- It saves disk space and makes transfers faster.
- Common tools: **gzip**, **bzip2**, **xz**, and modern **zstd**.

The **`file` command** in Linux is ==a command-line utility used to **determine the type and format of a file** by inspecting its actual data rather than relying on its filename extension==.
## ⚙️ **Common Compression Tools**

|Tool|Extension|Speed|Compression Ratio|Best Use|
|---|---|---|---|---|
|**gzip**|`.gz`|Fast|Good|Everyday logs, backups|
|**bzip2**|`.bz2`|Slower|Better|Source code archives|
|**xz**|`.xz`|Slowest|Best|Long-term storage, software distribution|
|**zstd**|`.zst`|Very fast|High|Modern backups, system compression|
|**zip**|`.zip`|Fast|Good|Cross-platform sharing (Windows/macOS)|
## 🔑 **Essential Commands**

### **gzip**
bash
```
gzip file.txt        # compress → file.txt.gz
gunzip file.txt.gz   # decompress
```
### **bzip2**
bash
```
bzip2 file.txt       # compress → file.txt.bz2
bunzip2 file.txt.bz2 # decompress
```
### **xz**
bash
```
xz file.txt          # compress → file.txt.xz
unxz file.txt.xz     # decompress
```
### **zstd**
bash
```
zstd file.txt        # compress → file.txt.zst
unzstd file.txt.zst  # decompress
```

the commands below: 
- These commands let you **view the contents of compressed files directly** without manually decompressing them first.
- They behave like `cat`, but for compressed formats.
    
## ⚙️ **Commands Overview**

|Command|Format Supported|Example|Purpose|
|---|---|---|---|
|**zcat**|`.gz` (gzip)|`zcat file.txt.gz`|Displays contents of a gzip-compressed file.|
|**bzcat**|`.bz2` (bzip2)|`bzcat file.txt.bz2`|Displays contents of a bzip2-compressed file.|
|**xzcat**|`.xz` (xz)|`xzcat file.txt.xz`|Displays contents of an xz-compressed file.|