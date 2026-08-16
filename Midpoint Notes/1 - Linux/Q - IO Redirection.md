![[Pasted image 20260715135454.png]]
- **I/O Redirection** allows you to change the default input/output of commands.
- Normally:
    - **Input** comes from the keyboard.
    - **Output** goes to the terminal.
- With redirection, you can send output to files, read input from files, or chain commands together.

### Redirect STDOUT(Standard Output)
- **STDOUT** is the default output stream — normally printed to your terminal.
- Redirection lets you send this output somewhere else (like a file).
## ⚙️ **Redirecting STDOUT Operators**

|Operator|Purpose|Example|
|---|---|---|
|**>**|Redirects STDOUT to a file (overwrite).|`ls > files.txt`|
|**>>**|Redirects STDOUT to a file (append).|`echo "new line" >> notes.txt`|
|**1>**|Explicitly redirects STDOUT (same as `>`).|`ls 1> list.txt`|
![[Pasted image 20260715135735.png]]

### Redirect STDERR(Standard Error)
- **STDERR** is the stream where commands send error messages.
- By default, errors print to the terminal separately from normal output (STDOUT).
- Redirecting STDERR lets you capture or manage errors just like regular output.
## ⚙️ **Redirecting STDERR Operators**

|Operator|Purpose|Example|
|---|---|---|
|**2>**|Redirects STDERR to a file (overwrite).|`ls /fakepath 2> errors.txt`|
|**2>>**|Redirects STDERR to a file (append).|`ls /fakepath 2>> errors.txt`|
|**2>&1**|Combines STDERR with STDOUT into one stream.|`command > output.txt 2>&1`|
|**2>/dev/null**|Discards errors (sends them to “black hole”).|`ls /fakepath 2>/dev/null`|
![[Pasted image 20260715140019.png]]


---
### Command Line Pipes
![[Pasted image 20260715140420.png]]
- A **pipe** (`|`) takes the **STDOUT** of one command and sends it as the **STDIN** of another.
- This allows you to build **command pipelines** that process data step by step.
    
## ⚙️ **Basic Usage**
bash
```
command1 | command2
```
- Output of `command1` becomes input for `command2`.
Example:
bash
```
ls | grep ".txt"
```
→ Lists files, then filters only those ending in `.txt`.


## 🔑 **Common Pipe Workflows**

| Workflow                 | Example         | Purpose      |                                             |        |                                             |
| ------------------------ | --------------- | ------------ | ------------------------------------------- | ------ | ------------------------------------------- |
| **Filter output**        | `dmesg`         | `grep usb`   | Show only USB-related kernel messages.      |        |                                             |
| **Count results**        | `ls`            | `wc -l`      | Count how many files are in a directory.    |        |                                             |
| **Sort results**         | `cat names.txt` | `sort`       | Sort names alphabetically.                  |        |                                             |
| **Chain multiple pipes** | `cat logfile`   | `grep error` | sort                                        | `uniq` | Find, sort, and deduplicate error messages. |
| **Paginate output**      | `ps aux`        | `less`       | View long process lists one page at a time. |        |                                             |

## 📝 **Examples in Action**

- **Search running processes**:
    bash
    ```
    ps aux | grep firefox
    ```

- **Count log errors**:
    bash
    ```
    zcat logfile.gz | grep "ERROR" | wc -l
    ```
    
- **Find top 10 largest files**:
    bash
    ```
    du -sh * | sort -rh | head -10
    ```

    ![[Pasted image 20260715141221.png]]
- **tee** reads from **STDIN** and writes simultaneously to **STDOUT** and one or more files.
- It’s like a “T-junction” in plumbing — the stream goes both ways.
    

## ⚙️ **Basic Usage**

bash
```
command | tee file.txt
```
- Runs `command`, saves its output into `file.txt`, and still displays it on the terminal.
    
## 🔑 **Common Options**

|Option|Purpose|Example|
|---|---|---|
|**tee**|Write output to file and terminal|`ls|tee files.txt`|
|**tee -a**|Append instead of overwrite|`echo "new line"|tee -a notes.txt`|
|**tee multiple files**|Write to multiple files|`ls|tee list1.txt list2.txt`|