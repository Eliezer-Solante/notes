### Searching for Files and Directories

![[Pasted image 20260715133128.png]]

---
### Grep

![[Pasted image 20260715133611.png]]

![[Pasted image 20260715134144.png]]



## 📦 **What grep Does**

- **grep** stands for **Global Regular Expression Print**.
- It searches through files (or input streams) for lines that match a **pattern**.
- By default, it prints matching lines to the terminal.

## 🔑 **Common Options**
| Option      | Purpose                              | Example                          |                   |
| ----------- | ------------------------------------ | -------------------------------- | ----------------- |
| **grep -i** | Case-insensitive search              | `grep -i "error" logfile.txt`    |                   |
| **grep -r** | earch in directories                 | `grep -r "main" /home/user/code` |                   |
| **grep -n** | Show line numbers                    | `grep -n "TODO" script.py`       |                   |
| **grep -v** | Show lines _not_ matching            | `grep -v "success" logfile.txt`  |                   |
| **grep -E** | Use extended regex                   | `grep -E "cat                    | dog" animals.txt` |
| **grep -w** | Match whole words only               | `grep -w "cat" animals.txt`      |                   |
| **grep -A** | Show _N_ lines **after** each match  | `grep -A 3 "error" logfile.txt`  |                   |
| **grep -B** | Show _N_ lines **before** each match | `grep -B 2 "error" logfile.txt`  |                   |


The **`file` command** in Linux is ==a command-line utility used to **determine the type and format of a file** by inspecting its actual data rather than relying on its filename extension==.