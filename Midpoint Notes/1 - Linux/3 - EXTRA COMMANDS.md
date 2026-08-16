- Copying files owned by a specific user while preserving their directory structure. Let’s break it down step by step:
```
find /home/usersdata/ -type f -user jim -exec cp --parents {} /news \;

```
- `-e YYYY-MM-DD` - sets user expiry date
    - `sudo chage -E YYYY-MM-DD username`
    - `sudo chage -l john` - to verify the expiry

- `-M` - sets user without directory
- `-s /sbin/nologin` sets user without interactive shell 

- To disable direct SSH root login
    - edit `/etc/ssh/sshd_config` file to `PermitRootLogin  no`  
    - restart the sshd service `sudo systemctl restart sshd`
    - verify by exiting the server, then login using `root@server_hostname`

Viewing and Inspecting Content

Before changing file data, these core tools help preview and inspect text content:

- `cat`: Outputs the entire file content to the terminal.

- `less`: Opens a scrollable interface to view massive text files page by page.

- `head`: Displays only the first 10 lines of a file by default.

- `tail`: Displays only the last 10 lines of a file.

- `tail -f`: Monitors and updates changes to a file in real-time. 

Content Transformation and Editing

These utilities act on data streams to substitute strings, delete lines, or reformat files: []

---

- ==`sed`: Performs advanced search-and-replace transformations using regular expressions.==
       
    1. Replace the First Match in Every Line 
By default, `sed` only replaces the **very first** occurrence of a word on each line. 

bash

```
sed 's/cat/dog/' pets.txt
```

Use code with caution.

- **Result**: Changes the first "cat" to "dog" on every line. [[1]]

2. Replace All Matches (Global Substitution) 
Add the `g` flag at the end to change **every** instance of the word in the entire file. [[1]

bash

```
sed 's/cat/dog/g' pets.txt
```

Use code with caution.

- **Result**: Changes every single "cat" to "dog". 
3. Delete Specific Lines []

Use the `d` command to delete lines matching a word or a specific line number. 
bash

```
# Delete all lines containing the word "error"
sed '/error/d' log.txt

# Delete exactly line number 5
sed '5d' log.txt
```

Use code with caution.

4. Case-Insensitive Matching

Add the `I` flag to match text regardless of capitalization.

bash

```
sed 's/apple/orange/gI' fruit.txt
```

Use code with caution.

- **Result**: Replaces "apple", "Apple", and "APPLE" with "orange". [[1]]


Important: Safe Testing vs. Permanent Saving

By default, running `sed` **only prints the output to your terminal screen**. It does not change the actual file. This makes it safe to test your commands first.

- **To save changes to a new file:** Use the `>` operator.
    
    bash
    
    ```
    sed 's/old/new/g' original.txt > updated.txt
    ```
    
    Use code with caution.
    
- **To overwrite the original file directly:** Use the `-i` (in-place) flag.
    
    bash
    
    ```
    sed -i 's/old/new/g' original.txt
    ```

---

    
    Use code with caution.

- `awk`: Extracts specific fields, manipulates text columns, and generates data reports.

- `tr`: Translates or deletes specific characters (e.g., converting lowercase to uppercase).

- `cut`: Slices out selected sections or columns of text from each line.

- `paste`: Merges lines of text from multiple files side-by-side. [[1]]

==Searching, Sorting, and Filtering==

==Use these commands to group, locate, and clean up internal information:==

- ==`grep`: Extracts and displays only the lines that match a specific text pattern.==

- ==`sort`: Organizes the lines of a file alphabetically or numerically.==

- ==`uniq`: Removes or reports adjacent duplicate lines within a pre-sorted file.==

- ==`wc`: Counts the total lines, words, and byte size of the target content.== 

File Comparison and Diffing

When analyzing discrepancies across multiple versions of file content:

- `diff`: Compares two files line-by-line and outputs the exact discrepancies.

- `cmp`: Checks two files byte-by-byte to spot the first character difference.

- `comm`: Compares two sorted files and outlines unique vs. shared data lines. [


# Crontab Restrictions
- ### 🔒 How Crontab Restrictions Work

- `cron.allow` = /etc/cron.allow

    - If this file exists, _only_ users listed inside can use `crontab`.
        
    - Example:
        
        Code
        
        ```
        root
        admin
        deploy
        ```
        
    - Anyone not listed will see: _“You are not allowed to use this program (crontab)”_.
        
- `cron.deny` = /etc/cron.deny
    
    - If `cron.allow` does not exist, this file blocks specific users.
        
    - Example:
        
        Code
        
        ```
        guest
        testuser
        tempworker
        ```
        
    - All other users remain allowed.


    # Firewalld
- Install 
```
    sudo apt install firewalld
    or 
    sudo yum install firewalld

```
- Check status
```
firewall-cmd --state
```
- Enable firewalld
```
    sudo systemctl enable --now firewalld
```
- - List active zones
```
firewall-cmd --get-active-zones
```
- Open a Port
```
    firewall-cmd --add-port=8080/tcp
```
- Make it Permanent
```
    firewall-cmd --permanent --add-port=8080/tcp
    firewall-cmd --reload
```
- Block an IP
```
    firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="192.168.1.100" drop'

```

- Make a zone public
```
sudo firewall-cmd --set-default-zone=public
```
    or
Use this option to move a specific network card (e.g., `eth0`) into the public zone.

1. **Change the zone permanently:**
    
    bash
    
    ```
    sudo firewall-cmd --permanent --zone=public --change-interface=eth0
    ```
    
    Use code with caution.
    
2. **Reload the firewall to apply:**
    
    bash
    
    ```
    sudo firewall-cmd --reload
    ```
    
    Use code with caution.



    You can achieve that with a simple combination of commands using `ls`, `sort`, and **redirection**:

bash

```
ls | sort > files_list.txt
```

### 🔍 Breakdown:

- `ls` → Lists all files and directories in the current working directory.
    
- `sort` → Arranges the output alphabetically by default.
    
- `>` **redirection** → Sends the sorted output into a file named `files_list.txt`. If the file doesn’t exist, it will be created; if it does exist, it will be overwritten.
    

### 📌 Variations:

- To include **hidden files** (those starting with `.`):
    
    bash
    
    ```
    ls -a | sort > files_list.txt
    ```
    
- To ensure only **files** (not directories) are listed:
    
    bash
    
    ```
    find . -maxdepth 1 -type f | sort > files_list.txt
    ```
    
- To **append** instead of overwrite:
    
    bash
    
    ```
    ls | sort >> files_list.txt
    ```