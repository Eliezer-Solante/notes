![[Pasted image 20260731112605.png]]

![[Pasted image 20260731112911.png]]


### Manual
![[Pasted image 20260731113627.png]]
![[Pasted image 20260731113911.png]]
![[Pasted image 20260731113935.png]]
to create a network, run: `docker network create <network-name>`

### Transition to Compose
![[Pasted image 20260731114303.png]]

![[Pasted image 20260731114400.png]]
![[Pasted image 20260731114500.png]]
![[Pasted image 20260731114742.png]]
![[Pasted image 20260731114913.png]]

==To set the container explicitly==
- use `container_name: httpd`
```yaml
services:
  httpd:
    image: httpd:latest
    container_name: httpd #HERE
    ports:
      - "6200:80"
    volumes:
      - /opt/sysops:/usr/local/apache2/htdocs    
```


![[Pasted image 20260731130802.png]]
`docker compose up` → Compose auto-discovers the file in the current directory (looking for `compose.yaml`, `compose.yml`, `docker-compose.yaml`, or `docker-compose.yml`, in that order).
`docker compose -f <file> up` → you're overriding that auto-discovery and telling it exactly which file to use instead.

One small nuance worth knowing: `up` isn't really "automatic" in the sense of always working — if there's no file with one of those default names in the current directory, plain `docker compose up` will fail with an error like `no configuration file provided`. So `-f` isn't just an alternative, it's also your fallback whenever the file doesn't match the default naming or isn't in your current folder.

By using ==Compose== method, network will be automatically created



![[Pasted image 20260731140732.png]]
If ever you already removed the images and you want to use it again, you just need to replace the `image: vote` with `build:./vote` or `build: <directory-of-the-image>`
and will be built again on running compose, downloaded from the cache.


Please click on the links below for further reference:

[https://docs.docker.com/compose/](https://docs.docker.com/compose/)

[https://docs.docker.com/engine/reference/commandline/compose/](https://docs.docker.com/engine/reference/commandline/compose/)

[https://github.com/dockersamples/example-voting-app](https://github.com/dockersamples/example-voting-app)