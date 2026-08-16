
#### **GUI** 
- Graphical User Interface (Ubuntu Desktop) 
#### **CLI** 
- Command Line Interface (Linux Shell)

#### **Linux Shell** 
- is a program that allows text-based interaction between the user and the operating system. This interaction are carried out by typing commands into the interface and receiving the response in the same way. 

#### **Home Directory** 
- (ex. /home/michael)
- allows users to store their personal data in the form of files in directory
- represented by ***tilde*** symbol "[~]$" in the command line 

#### **Command and Arguments** 
 - The "**echo**" command is used to print a line of text on the screen 
 - An "**argument**" acts as an input to the command 
		ex. 
		
```
			[~]$ echo Hello 
			Hello
			[~]$
```

- where "**echo**" is the command to print, while "Hello" is an "argument" on what to print
    
- The "**uptime**" to print on how long the system is running for since the last reboot. This command does not need an argument to run.
    
- In the "**echo**" command use "**-n**" to print the same word "**Hello**" but without a trailing line afterward.
		ex.  
	
```
		[~]$ echo Hello
		Hello[~]$
```

#### **Command Types**
##### Internal or Built-in Commands
- echo, cd, pwd, set, e.t.c
- part of the shell itself
- Total of 30 commands 
##### External Commands
- mv, date, uptime, cp, uptime e.t.c
- binary programs or scripts 
- pre-installed with a package 
- created or installed by the user 

To determine what type of command you are using use "type" command.
    ex. 
```
			[~]$ type echo
			echo is a shell built-in
			[~]$
```
```
    		[~]$ type mv
			mv is hashed (/bin/mv)
			[~]$
```