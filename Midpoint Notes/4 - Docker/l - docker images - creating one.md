


 ![[Pasted image 20260731091916.png]]

 ![[Pasted image 20260731092133.png]]

 ![[Pasted image 20260731092228.png]]

 ![[Pasted image 20260731092521.png]]

 ![[Pasted image 20260731092614.png]]


 ![[Pasted image 20260731092724.png]]

Note: Run `ducker run -it ubuntu` to initialize the steps 


<mark style="background: #ABF7F7A6;">To make a lite version of an image use this:</mark>
ex.
In the `webapp-color` directory, run the `ls -l` command to list the Dockerfile and other files.

And modify Dockerfile to use `python:3.14-alpine` image and then build using 
`docker buildx build -t webapp-color:lite .`


EXAMPLE SITUATION:

As per recent requirements shared by the Nautilus application development team, they need custom images created for one of their projects. Several of the initial testing requirements are already been shared with DevOps team. Therefore, create a docker file `/opt/docker/Dockerfile` (please keep `D` capital of Dockerfile) on `App server 3` in `Stratos DC` and configure to build an image with the following requirements:  
  

a. Use `ubuntu:24.04` as the base image.  
  

b. Install `apache2` and configure it to work on `6300` port. (do not update any other Apache configuration settings like document root etc).

```Dockerfile
FROM ubuntu:24.04

RUN apt update && \
    apt install -y apache2 && \
    rm -rf /var/lib/apt/lists/*

RUN sed -i 's/Listen 80/Listen 6300/' /etc/apache2/ports.conf && \
    sed -i 's/<VirtualHost \*:80>/<VirtualHost *:6300>/' /etc/apache2/sites-enabled/000-default.conf

EXPOSE 6300

CMD ["apache2ctl", "-D", "FOREGROUND"]
```
### Explanation

- **`FROM ubuntu:24.04`** — uses the exact base image required.
- **`RUN apt update && apt install -y apache2`** — installs Apache2 non-interactively; cleans up apt cache in the same layer to keep image size down.
- **Two `sed` commands** — change only the **port** Apache listens on (`ports.conf`) and the matching **VirtualHost** block (`000-default.conf`). Nothing else is touched — document root, modules, and all other settings stay at their defaults, per the requirement.
- **`EXPOSE 6300`** — documents the port (doesn't publish it by itself, but good practice).
- **`CMD ["apache2ctl", "-D", "FOREGROUND"]`** — runs Apache in the foreground so it stays as the container's main process (PID 1). This is important: without `FOREGROUND`, Apache would background itself and the container would immediately exit since Docker requires a foreground process to keep running.

### Build and run
bash
```bash
docker build -t apache-6300:v1 .
docker run -d -p 6300:6300 --name apache_container apache-6300:v1
```

### Verify
bash
```bash
docker ps
curl http://localhost:6300/
```
You should get Apache's default "It works!" page back on port 6300.


SAMPLES

### Node.js — `/node_app/Dockerfile`

dockerfile

```dockerfile
FROM node:20-alpine

WORKDIR /node_app

COPY package.json ./

RUN npm install

COPY . .

EXPOSE 3004

CMD ["node", "server.js"]
```

### Python — `/python_app/Dockerfile`

dockerfile

```dockerfile
FROM python:3.12-slim

WORKDIR /python_app

COPY src/requirements.txt ./

RUN pip install --no-cache-dir -r requirements.txt

COPY . .

EXPOSE 8085

CMD ["python", "src/server.py"]
```

### Quick reference

||Node.js|Python|
|---|---|---|
|Base image|`node:20-alpine`|`python:3.12-slim`|
|Dependency file|`package.json`|`requirements.txt`|
|Install command|`npm install`|`pip install -r requirements.txt`|
|Entry script|`server.js`|`server.py`|
|Exposed port|`3004`|`8085`|
|Build path|`/node_app`|`/python_app`|