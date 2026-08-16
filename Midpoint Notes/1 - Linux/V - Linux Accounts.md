![[Pasted image 20260715191549.png]]

![[Pasted image 20260715191720.png]]

Linux User 
- info can be seen using `cat /etc/passwd`
- has a unique identifier called UID (User ID)

Linux Group
- info can be seen using `cat /etc/group`
- has a unique identifier called GID (Group ID)

![[Pasted image 20260715192212.png]]
![[Pasted image 20260715192415.png]]
![[Pasted image 20260715192525.png]]


![[Pasted image 20260715192654.png]]


![[Pasted image 20260715193012.png]]
![[Pasted image 20260715193653.png]]
## 🧱 Structure of a Sudoers Entry

Each line follows this pattern:
Code

```
user host=(runas_user) command
```

Here’s what each part means

|Field|**Description**|Example|
|---|---|---|
|1️⃣ **User or Group**|Who gets the privilege|`bob`, `%sudo` (group)|
|2️⃣ **Host**|Where they can run it|`localhost`, `ALL` (any host)|
|3️⃣ **Runas User**|Who they can act as|`ALL` (default = any user)|
|4️⃣ **Command**|What they can execute|`/bin/ls`, `ALL` (any command)|
## 🧠 Example Breakdown (from the image)

Let’s decode a few lines:
- `root ALL=(ALL:ALL) ALL` → Root can run any command on any host as any user.
- `%admin ALL=(ALL) ALL` → Members of the `admin` group can run any command as any user.
- `bob ALL=(ALL:ALL) ALL` → Bob can run any command anywhere, just like root.
- `sarah localhost=/usr/bin/shutdown -r now` → Sarah can only run the reboot command (`shutdown -r now`) on her local machine.
