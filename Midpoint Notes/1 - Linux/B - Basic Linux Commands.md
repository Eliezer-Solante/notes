Goal: Create a directory structure
![[Pasted image 20260713172919.png]]
- run `pwd` or ==present working== directory command to check the current directory you are in.

- run `ls` or ==list contents== to check the list the contents that are inside that current directory 


---

### Creating and Navigating Directories 

- ==to make a directory== - 
```
[~]$ mkdir Asia
```

- ==to make multiple directories== -
```
[~]$ mkdir Asia Europe Africa America 
```
    use spaces to separate each directory

- run `cd [directory name]` or change directory to change your location you want to work in and run `pwd` to check whether you're in the right directory.
```
[~]$ cd Asia
[~/Asia]$
```

- ==to make a directory inside a directory without getting inside that directory== - 
```
[~/Asia]$ mkdir India/Mumbai
```

- ==to make both the parent and child directory== -
```
[~/Asia]$ mkdir -p India/Mumbai
```
where `-p` or <mark style="background: #FFF3A3A6;">Parents</mark> flag automatically creates the parent folder `India` and its child directory which is the `Mumbai`.

- ==to go back to the home directory directly from a sub-directory== - 
```
[~/Asia/India]$ cd
[~]$

or

[~/Asia/India]$ cd /home/micahel
[~]$

```

- ==to go back to the previous directory== -
```
[~/Asia/India]$ cd ..
[~/Asia]$
```

---
### Absolute and Relative Path 
![[Pasted image 20260714094623.png]]
==Absolute Path== - locating a file or a directory from the root directory( ==/== )
```
[~]$ cd /home/michael/Asia
```
meaning it iterates each directory until the desired directory(/Asia) is reached

==Relative Path== - locating a file or a directory from the parent directory(relative)
```
[~/]$ cd Asia
```


##### Pro Tip - "pushd" and "popd" commands
- `pushd` and `popd` are shell commands that let you **navigate directories using a stack** — they remember where you’ve been.
    
- `pushd` works like `cd`, but it also saves your current directory on the stack before moving.
    
- `popd` takes you back to the last directory saved on the stack, not just the parent folder like `cd ..`.
    
- This makes switching between multiple directories faster, without needing to type full paths or constantly check with `pwd`.

```
# Start at home
pwd
# /home/michael

# Push Asia/India/Mumbai onto stack
pushd Asia/India/Mumbai
# /home/michael/Asia/India/Mumbai /home/michael

# Push Europe/UK onto stack
pushd ../../Europe/UK
# /home/michael/Europe/UK /home/michael/Asia/India/Mumbai /home/michael

# Pop back to Mumbai
popd
# /home/michael/Asia/India/Mumbai /home/michael

# Pop back to home
popd
# /home/michael

```


---
![[Pasted image 20260714104405.png]]
<mark style="background: #FFB86CA6;">Note: the goal is to change/edit the left figure to become identical to the right by using commands.</mark>

1. First is to move the directory "Morocco" from Europe to Africa directory by using `mv`  or <mark style="background: #FFF3A3A6;">Move file</mark> command.
`mv` command requires two arguments: source file/directory and destination directory
```
[~]$ mv [source file/directory]  [destination directory]
```
so, (<mark style="background: #D2B3FFA6;">Absolute Path</mark>)
```
[~]$ mv /home/michael/Europe/Morroco /home/michael/Africa
```
or, if assuming you are already on `/home/michael` directory, then use <mark style="background: #D2B3FFA6;">Relative path</mark>,
```
[~]$ mv Europe/Morroco Africa
```


2. Second is to fix the directory spelling`Munbai` to `Mumbai` by renaming it using the same `mv` command.
```
[~]$ mv Asia/Indi/Munbai Asia/Indi/Mumbai
```

3. Third is to copy the `City.txt` file from `MumbaI` directory to the `Cairo` directory by using `cp` or <mark style="background: #FFF3A3A6;">Copy file</mark> command.
```
[~]$ cp [source file/directory]  [destination directory]
```
so, 
```
[~]$ cp Asia/Indi/Munbai/City.txt Africa/Egypt/Cairo
```

4. Lastly is to delete the `Tottenham.txt` file from the `London` directory by using `rm` or <mark style="background: #FFF3A3A6;">Remove file</mark> command.
```
[~]$ rm Europe/UK/London/Tottenham.txt
```

NOTE: If you want to copy `cp` or remove `rm` a <mark style="background: #ABF7F7A6;">DIRECTORY</mark>, you need to use the `-r` or <mark style="background: #FFF3A3A6;">Recursive</mark> flag, because the `cp` and `rm` are only for files.
- ex. if you want to remove the `London` directory/folder:
```
[~]$ rm -r Europe/UK/London
```

##### Working with Files and Directories
![[Pasted image 20260714134707.png]]

- The `cat` or <mark style="background: #FFF3A3A6;">Concatenate</mark> command is primarily designed to **read, display, create, and combine text files** directly from the command line.

- The `touch` or <mark style="background: #FFF3A3A6;">Touch</mark> command is a command that creates new, empty files and updates the access and modification timestamps of existing files.

- The `>` or <mark style="background: #FFF3A3A6;">Output redirection</mark> operator sends the output of a command to a file instead of the screen, overwriting the file if it exists.
        ex. To save a list of files in your current directory to a text file, use `ls > file.txt`.
        
- The `<` or <mark style="background: #FFF3A3A6;">Input redirection</mark> operator forces a command to read its input from a file instead of the keyboard.
        ex. Instead of typing text directly into the command, you can use `<` to feed a file into a tool like `wc` (word count): `wc -w < document.txt`.

Reading the content of a file, for example the `City.txt` in the Mumbai directory (This case, `cat` is used to <mark style="background: #FFB86CA6;">read and display</mark> the `City.txt`  content)
```
[~]$ cat Asia/India/Mumbai/City.txt
Mumbai
```
but, because earlier we copied the `City.txt` to the `Cairo` folder, the content of it was also copied, so we need to change the content of the `City.txt` to the appropriate content,
```
[~]$ cat Africa/Egypt/Cairo/City.txt
Cairo
```
Once this command is issued the prompt will immediately ask for input (Lines or text) to be redirected towards the `City.text` content and replace it. After inputting any contents to the prompt, press `ctrl + d` to exit the prompt

![[Pasted image 20260714144312.png]]
These <mark style="background: #FFF3A3A6;">Pagers</mark> are for browsing the contents of a `txt` file that have multiple lines

- `ls -l` (Long List)
```
[~]$ ls -l
total 8 
drwxr-xr-x 2 user user 4096 Jul 14 13:54 documents/ 
drwxr-xr-x 2 user user 4096 Jul 14 13:54 projects/
```
==displays the contents of a directory in a **long listing format**==. Instead of just showing the names of files and folders, it provides a detailed, column-by-column breakdown of their metadata

- `ls -a` (List all files including hidden files)
```
[~]$ ls -a
./ ../ .gitconfig .zshrc documents/ projects/
```

- `ls -lt` (long list files in order created) newest to oldest
```
[~]$ ls -lt
total 8 
-rw-r--r-- 1 user user 0 Jul 14 15:01 sample2.txt 
drwxr-xr-x 2 user user 4096 Jul 14 13:54 documents/ 
drwxr-xr-x 2 user user 4096 Jul 14 13:54 projects/
```

- `ls l-ltr` (Long list files in the reverse order created) oldest to newest
```
[~]$ ls -ltr
drwxr-xr-x 2 user user 4096 Jul 14 13:54 projects/ 
drwxr-xr-x 2 user user 4096 Jul 14 13:54 documents/ 
-rw-r--r-- 1 user user 0 Jul 14 15:01 sample2.txt
```


---


