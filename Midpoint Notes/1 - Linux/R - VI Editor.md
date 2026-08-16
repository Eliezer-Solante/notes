![[Pasted image 20260715152701.png]]
- **VI Editor** is a terminal-based text editor.
- **Pre-installed** on most Linux/Unix systems.
- **Modal editing**: behavior changes depending on the mode.

## **Modes in VI**

|Mode|Purpose|How to Enter|
|---|---|---|
|**Command Mode**|Default mode; navigate, delete, copy, issue commands.|Starts automatically when you open a file.|
|**Insert Mode**|Enter text into the file.|Press `i`, `a`, `o`, or `O`. Exit with `Esc`.|
|**Last Line Mode**|Execute commands (save, quit, search).|Type `:` from command mode.|


![[Pasted image 20260715153226.png]]

![[Pasted image 20260715153451.png]]

![[Pasted image 20260715153545.png]]

![[Pasted image 20260715153733.png]]



![[Pasted image 20260715154409.png]]


![[Pasted image 20260715154529.png]]




## ⚙️ **Command Mode (default mode)**

This is where you **navigate, delete, copy, and issue editing commands**.

|Command|Purpose|
|---|---|
|**h**|Move left|
|**j**|Move down|
|**k**|Move up|
|**l**|Move right|
|**0**|Beginning of line|
|**$**|End of line|
|**G**|Last line of file|
|**:n**|Go to line number _n_|
|**dd**|Delete current line|
|**yy**|Copy current line|
|**p**|Paste copied text|
|**u**|Undo last change|
|**Ctrl+r**|Redo undone change|

## ✍️ **Insert Mode**

This is where you actually **type text into the file**. You enter it from Command Mode and exit with `Esc`.

|Command|Purpose|
|---|---|
|**i**|Insert before cursor|
|**a**|Append after cursor|
|**o**|Open new line below|
|**O**|Open new line above|

## 📜 **Last Line Mode**

This is where you **execute commands** like saving, quitting, or searching. Enter it by typing `:` from Command Mode.

|Command|Purpose|
|---|---|
|**:w**|Save file|
|**:q**|Quit|
|**:wq**|Save and quit|
|**:q!**|Quit without saving|
|**/%word**|Search forward for “word”|
|**:%s/old/new/g**|Replace all “old” with “new”|
### VIM
![[Pasted image 20260715154900.png]]
