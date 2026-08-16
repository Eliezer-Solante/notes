`Docker run`

To create a custom container 
   use: `docker run -d -p 38282:8080 --name blue-app kodekloud/simple-webapp:blue`
            where:
                 `-d` - flag for detached
                 `-p` - flag to assign port
                 `--name` - flag for custom container name
                 `blue-app` - name
                 `kodekloud/simple-webapp:blue` - the image
                 
![[Pasted image 20260731135257.png]]
![[Pasted image 20260730162913.png]]

![[Pasted image 20260730162621.png]]
1- display running container
2- stop running container
3- check if still running -all
4- display containers history
5- remove container

removing an image 
![[Pasted image 20260730162649.png]]
Note: before removing an image, make sure that no container is running on that image 

pulling an image
![[Pasted image 20260730164319.png]]

You cannot run an OS image in a docker, it exits right away
![[Pasted image 20260730164523.png]]

![[Pasted image 20260730164609.png]]

Detach and Re-attach containers
![[Pasted image 20260730164759.png]]

==Clear out the system==
 - use `docker system prune`

 ==Inspect a container's port config==
 - `docker port <container_name_or_id>`

==Copy a file from host to a container==
-  `docker cp /path/on/host/file.txt container_name:/path/in/container/`
    - ex. `docker cp myfile.txt my_container:/opt/myfile.txt`

==Copy a file from a container to the host== 
- `docker cp container_name:/path/in/container/file.txt /path/on/host/`
    - ex. `docker cp my_container:/opt/myfile.txt .`

==Export an image to a `.tar` file==
- `docker save -o output.tar image_name:tag`
    - `docker save -o httpd.tar httpd:latest`

==Load an image archive==
- `docker load -i image.tar`

==Retagging an image==
- `docker tag source_image:tag new_image:tag`
    - `docker tag httpd:latest httpd:v1.0`

==Giving permissions to use docker commands without using `sudo`==
- `sudo usermod -aG docker username`

==Creating a docker image from a container==
- `docker commit container_name new_image_name:tag`
    - `docker commit my_container my-custom-httpd:v1`

==Entering a container's bash/Access the container==
- `docker exec -it kkloud bash`

    Example Situation 
        - a. Install `apache2` in `kkloud` container using `apt` that is running on `App Server 2` in `Stratos Datacenter`. 
        - b. Configure Apache to listen on port `6100` instead of default `http` port. Do not bind it to listen on specific IP or hostname only, i.e it should listen on localhost, 127.0.0.1, container ip, etc. 
        - c. Make sure Apache service is up and running inside the container. Keep the container in running state at the end.

### ==a. Install Apache2 in the `kkloud` container==

First, access the container:
bash
```bash
docker exec -it kkloud bash
```

Then update package lists and install Apache2:
bash
```bash
apt update
apt install -y apache2
```
### ==b. Configure Apache to listen on port 6100==

Still inside the container, edit the ports configuration file:
bash
```bash
vi /etc/apache2/ports.conf
```

Change the line:
```
Listen 80
```
to:
```
Listen 6100
```
**Important:** Do **not** specify an IP like `Listen 127.0.0.1:6100` — just `Listen 6100` on its own binds to all interfaces (`0.0.0.0`), which satisfies the requirement to not restrict it to a specific IP/hostname.

Also update the default virtual host so it responds correctly on this port:
bash
```bash
vi /etc/apache2/sites-enabled/000-default.conf
```
Change:
```
<VirtualHost *:80>
```
to:
```
<VirtualHost *:6100>
```

### ==c. Start Apache and keep it running==
Inside containers, `systemctl` often isn't available (no init system), so start Apache directly:
bash
```bash
service apache2 start
```
or
bash
```bash
apache2ctl start
```
**
Verify it's running:**
bash
```bash
service apache2 status
```

or check the process directly:
bash
```bash
ps aux | grep apache2
```

**Verify it's listening on the right port:**
bash
```bash
apt install -y net-tools   # if netstat isn't available
netstat -tulnp | grep 6100
```
You should see it bound to `0.0.0.0:6100`, not `127.0.0.1:6100`.

**Test locally inside the container:**
bash
```bash
curl http://localhost:6100/
```

Exit the container shell (but leave the container running):
bash
```bash
exit
```

==Installing docker on Centos==
### 1. Remove old versions (if any)

```bash
sudo yum remove docker docker-client docker-client-latest docker-common \
    docker-latest docker-latest-logrotate docker-logrotate docker-engine
```
### 2. Install required packages
```bash
sudo yum install -y yum-utils
```
### 3. Add the Docker repository
```bash
sudo yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
```
### 4. Install Docker Engine
```bash
sudo yum install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```
If you hit a dependency conflict with `containerd.io` (common on some CentOS 8 setups), run:

```bash
sudo yum install -y docker-ce --nobest
```
### 5. Start and enable Docker
```bash
sudo systemctl start docker
c
```
### 6. Verify it's working
```bash
sudo docker run hello-world
```
### 7. (Optional) Run Docker without sudo
```bash
sudo usermod -aG docker $USER
```

==Installing docker on ubuntu==
### 1. Remove old versions (if any)
```bash
sudo apt remove docker docker-engine docker.io containerd runc
```
### 2. Update package index and install prerequisites
```bash
sudo apt update
sudo apt install -y ca-certificates curl gnupg
```
### 3. Add Docker's official GPG key
```bash
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg
```
### 4. Add the Docker repository
```bash
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```
### 5. Install Docker Engine
```bash
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```
### 6. Verify it's working
```bash
sudo docker run hello-world
```
### 7. (Optional) Run Docker without sudo
```bash
sudo usermod -aG docker $USER
```







---



This command forces the removal of every single image stored on your system. 

bash

```
docker rmi $(docker images -q) -f
```

Use code with caution.

**How it works:**

- `docker images -q`: Lists only the unique numerical IDs of all images.
- `docker rmi`: Deletes the images passed from that list.
- `-f`: Forces removal (needed if an image is tagged in multiple repositories or linked to a stopped container).

---

If you want to safely clean up disk space without losing the images you are actively using, use the prune command instead. 

bash

```
docker image prune -a
```

Use code with caution.

**How it works:**

- `docker image prune`: Deletes all "dangling" images (layers left over from builds).
- `-a`: Also deletes any image that is not currently associated with a running or stopped container. 
---


### 1. Remove old versions (if any)

bash

```bash
sudo yum remove docker docker-client docker-client-latest docker-common \
    docker-latest docker-latest-logrotate docker-logrotate docker-engine
```

### 2. Install required packages

bash

```bash
sudo yum install -y yum-utils
```

### 3. Add the Docker repository

bash

```bash
sudo yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
```

### 4. Install Docker CE

bash

```bash
sudo yum install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

This last package, `docker-compose-plugin`, gives you the modern `docker compose` (v2, no hyphen) command as a Docker CLI plugin — this is the officially recommended way to get Compose now, rather than installing the standalone `docker-compose` binary.

### 5. Start and enable Docker

bash

```bash
sudo systemctl start docker
sudo systemctl enable docker
```

### 6. Verify installation

bash

```bash
sudo docker --version
sudo docker compose version
sudo docker run hello-world
```

### 7. (Optional) Run Docker without sudo

bash

```bash
sudo usermod -aG docker $USER
```

Then log out and back in (or run `newgrp docker`) for the group change to take effect.

---

#### If you specifically need the standalone `docker-compose` (v1-style) binary instead

Some older scripts still expect the hyphenated `docker-compose` command. You can install it separately:

bash

```bash
sudo curl -L "https://github.com/docker/compose/releases/latest/download/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose
docker-compose --version
```

#### Notes

- If you're on **CentOS Stream 9** or newer, the same steps work — just make sure `yum-utils` and the repo add step complete without errors.
- If `yum-config-manager` isn't found, install `dnf-plugins-core` instead (`sudo dnf install -y dnf-plugins-core`) since some newer CentOS/Stream releases use `dnf` naming.
- SELinux is enabled by default on CentOS — if containers have trouble with volume mounts, you may need `:z` or `:Z` flags on volume mounts, or adjust SELinux policies.