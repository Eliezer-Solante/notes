![[Pasted image 20260731144724.png]]

 ![[Pasted image 20260731145123.png]]
when accessing a remote docker engine, use the `-H` option and  specify the remote host's IP and Port 
ex. `-H=10.123.2.1:2376 --tlsverify run nginx`


 ![[Pasted image 20260731145452.png]]

 ![[Pasted image 20260731145648.png]]


 ![[Pasted image 20260731150739.png]]

**Docker Engine on Linux**

On Linux, Docker Engine runs natively — there's no VM in between. A container is just a regular Linux process, isolated using kernel features. This is why Docker on Linux is lightweight compared to Docker Desktop on Mac/Windows, which has to run a Linux VM under the hood since those OSes don't have the needed kernel primitives.

The engine itself is really a stack of cooperating daemons, not one monolithic program — that's what the diagram shows top to bottom.

**What's underneath the daemon**

`dockerd` (the daemon) doesn't actually create containers itself anymore — it delegates:

- **dockerd** — exposes the REST API, handles image builds, volumes, networks, and high-level orchestration. This is what your `docker` CLI commands actually talk to.
- **containerd** — a separate daemon (originally split out of Docker, now a CNCF project used by Kubernetes too) that manages the container lifecycle: pulling images, managing storage, starting/stopping containers.
- **runc** — the low-level OCI-compliant runtime that does the actual work of creating a container: setting up namespaces, cgroups, and executing the process. This is the piece that directly talks to the Linux kernel.
- **Linux kernel primitives**:
    - **Namespaces** — isolate what a process can _see_ (its own PID tree, network stack, mount points, hostname, etc.)
    - **cgroups** — limit what a process can _use_ (CPU, memory, I/O)
    - **Union filesystem** (overlay2 by default) — lets image layers stack on top of each other efficiently

So a "container" is really just a Linux process that runc launched with a specific combination of namespaces and cgroup limits applied — there's no container-specific kernel feature, it's a composition of general-purpose Linux tools.

**Remote Docker Engine**

By default, `dockerd` listens on a local Unix socket (`/var/run/docker.sock`), which only the local machine can use. To manage a remote Docker host, you point your CLI at it instead:

bash

```bash
export DOCKER_HOST=tcp://remote-host:2376
docker ps
```

Or per-command:

bash

```bash
docker -H tcp://remote-host:2376 ps
```

**Remote access policy — this is where security matters a lot**

The Docker daemon socket is effectively root access to the host — if you can talk to it, you can mount the host filesystem into a container and read/write anything. So exposing it remotely needs care:

- **Never expose the daemon on `tcp://` without TLS.** An unauthenticated Docker socket on the network is a full remote-root vulnerability — this is a very common real-world misconfiguration attackers scan for.
- **Use TLS client certificates** — the standard approach: generate a CA, server cert, and client cert, then start the daemon with `--tlsverify --tlscacert --tlscert --tlskey`, and connect with matching client certs. This restricts access to only clients holding a valid cert.
- **Prefer an SSH tunnel over raw TCP** — increasingly the recommended approach:

bash

```bash
  docker -H ssh://user@remote-host ps
```

This reuses your existing SSH keys/auth instead of managing a separate TLS PKI, and doesn't require opening port 2376 at all.

- **Firewall the daemon port** — if you must use TCP, restrict `2376` to trusted IPs only, never `0.0.0.0` open to the internet.
- **Context feature** — Docker CLI "contexts" let you save and switch between multiple remote/local engines cleanly:

bash

```bash
  docker context create myremote --docker "host=ssh://user@remote-host"
  docker context use myremote
```

**Bottom line:** the safest way to manage a remote Docker host is SSH (`ssh://`), not raw TCP — it avoids exposing the daemon socket over the network entirely while still giving you full remote control.

