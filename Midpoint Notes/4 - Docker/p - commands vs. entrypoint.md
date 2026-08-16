
![[Pasted image 20260731105247.png]]

![[Pasted image 20260731105400.png]]![[Pasted image 20260731105542.png]]

==to overwrite a CMD:==

bash
`docker run [options] <image_name> <command>`

Everything after the image name is treated as the command (and its arguments), and it completely replaces the image's default CMD.

Example
bash
`docker run -d --name my_container -p 8085:80 httpd:latest httpd-foreground`