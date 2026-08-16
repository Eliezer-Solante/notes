A ==**Docker registry**== is a storage and distribution system for Docker images — basically a place where images live so you can push them up and pull them down, rather than rebuilding or manually copying them between machines.
![[Pasted image 20260801224642.png]]

![[Pasted image 20260801223600.png]]![[Pasted image 20260801223621.png]]
![[Pasted image 20260801223639.png]]



![[Pasted image 20260801223713.png]]

![[Pasted image 20260801223752.png]]

![[Pasted image 20260801223846.png]]

![[Pasted image 20260801224233.png]]


![[Pasted image 20260801224358.png]]

#### to check pushed images, run `curl localhost:5000/v2/_catalog`
A Docker registry is a storage and distribution system for Docker images — basically a place where images live so you can push them up and pull them down, rather than rebuilding or manually copying them between machines.

## The basic pieces

**Registry** — the server that stores images (e.g., Docker Hub, or a self-hosted one). **Repository** — a collection of related images in a registry, usually different versions of the same app (e.g., `nginx`). **Tag** — a label for a specific version within a repository (e.g., `nginx:1.27`, `nginx:latest`).

Put together, an image reference looks like:

```
registry-host:port/namespace/repository:tag
```

For example:

```
myregistry.example.com:5000/myteam/myapp:v2.1
```

If you omit the registry host, Docker assumes Docker Hub.

## Common registries

- **Docker Hub** — the default public registry, free tier has pull rate limits for anonymous/free accounts
- **GitHub Container Registry (ghcr.io)** — integrates well if your code's already on GitHub
- **Amazon ECR, Google Artifact Registry, Azure Container Registry** — cloud-native options, good if you're already in that ecosystem
- **Self-hosted `registry:2`** — Docker's own open-source registry image, good for private/internal use

## Basic workflow

```bash
# Log in
docker login myregistry.example.com

# Tag your local image for the registry
docker tag myapp:latest myregistry.example.com/myteam/myapp:v1.0

# Push it
docker push myregistry.example.com/myteam/myapp:v1.0

# Pull it elsewhere
docker pull myregistry.example.com/myteam/myapp:v1.0
```

## Running your own registry

Quick and dirty (no auth, not for production):

```bash
docker run -d -p 5000:5000 --name registry registry:2
```

For anything real you'll want TLS and authentication — either via a reverse proxy (nginx/traefik with basic auth) or by mounting htpasswd credentials directly into the registry container.

Want me to go deeper on any part of this — self-hosting with auth/TLS, private registries in Kubernetes (imagePullSecrets), vulnerability scanning, or registry mirroring/caching for CI speed?