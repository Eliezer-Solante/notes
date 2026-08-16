
Inspect
![[Pasted image 20260730233612.png]]
use this command when you want to find details about a container
`docker inspect <container-name>`


Logs
![[Pasted image 20260730233753.png]]
`docker logs <container-id or name>`


to know the what is the host port and container port 
- first, run `docker ps` or `docker ps -a`
- `38282/8080` 
    - `38282` is the host port
    - `8080` is the container port


<mark style="background: #FFB86CA6;">NOTE: To see what base OS is used by a certain image run </mark>
`docker run <image> cat /etc/*release*`
