![[Pasted image 20260801220345.png]]

![[Pasted image 20260801220423.png]]

![[Pasted image 20260801220448.png]]
to see what subnet a network is on run  
- `docker network inspect <network-name>`



![[Pasted image 20260801220605.png]]

![[Pasted image 20260801220911.png]]

  ![[Pasted image 20260801221020.png]]




## Core model

- Each container gets its own **network namespace** by default — an isolated view of network interfaces, unless told otherwise.
- A **network driver** decides how containers connect: `bridge`, `host`, `none`, `overlay`, `macvlan`, `ipvlan`.
- Containers on the **same network** can reach each other freely by default — no built-in firewall between them. Isolation is achieved mainly by _not_ putting containers on the same network.

## Network drivers

|Driver|Scope|Use case|
|---|---|---|
|`bridge` (default)|Single host, private, NAT'd|General use; custom bridge networks add DNS-based service discovery by container name|
|`host`|Single host, no isolation|Container shares host's network stack directly — no port mapping, direct binding|
|`none`|Single host, no networking|Full network isolation — sandboxing, batch jobs|
|`overlay`|Multi-host|Connects containers across different Docker hosts (Swarm), via VXLAN tunnels|
|`macvlan`|Single host, LAN-level|Container gets its own MAC + IP, appears as a real device on the physical network|
|`ipvlan`|Single host, LAN-level|Like macvlan but shares one MAC address across containers|

## Creating and configuring networks

```bash
docker network create mynet                          # default bridge driver
docker network create --driver overlay my-overlay     # or -d
docker network create --internal secure-net           # no outbound access at all

docker network create \
  --driver bridge \
  --subnet 172.28.0.0/16 \
  --gateway 172.28.0.1 \
  --ip-range 172.28.5.0/24 \
  mynet
```

- `--driver` / `-d` — which driver to use (defaults to `bridge` if omitted)
- `--subnet` — explicit IP range, avoids collisions with your real LAN/VPC and is required for macvlan/ipvlan to match physical network addressing
- `--gateway` — gateway IP within the subnet
- `--ip-range` — narrows which portion of the subnet gets auto-assigned

## Attaching containers

```bash
docker run --network mynet --name app myimage
docker run --network mynet --ip 172.28.5.10 --name app myimage   # fixed IP (user-defined networks only)
docker network connect mynet myapp
docker network disconnect mynet myapp
```

## Port publishing (bridge networks only — `host` needs none)

```bash
docker run -p 8080:80 myimage       # host:container
docker run -P myimage                # auto-publish all EXPOSEd ports
```

`EXPOSE` in a Dockerfile is documentation only — `-p` is what actually opens the mapping.

## Isolating containers on one host

- **Separate networks per trust boundary** — the main tool. Only attach a container to more than one network if it genuinely needs to bridge them.
- **`--internal` networks** — group can talk internally, nothing reaches outside.
- **`icc: false`** in `daemon.json` — disables inter-container communication on the _default_ bridge specifically.
- Fine-grained per-port ACLs between containers on the same network aren't natively supported — that needs iptables/nftables rules or a service mesh.

## Restricting a single container's outbound access

```bash
docker run --network none myimage              # no networking at all
docker network create --internal isolated_net  # talk to siblings, not outside
```

## Reaching outside the host (usually automatic)

- Default `bridge` already allows outbound via NAT.
- If it's broken: check `--network none`/internal misconfig, DNS (`--dns 8.8.8.8`), or test raw connectivity with `docker exec -it <c> ping 8.8.8.8`.
- `--network host` removes NAT/isolation entirely — container uses host's network directly (Linux only, not Docker Desktop on Mac/Windows).

## Inspecting and debugging

```bash
docker network ls
docker network inspect mynet
docker network inspect mynet --format '{{json .IPAM.Config}}'
docker inspect myapp --format '{{json .NetworkSettings.Networks}}'
docker exec -it myapp ping db          # tests connectivity + DNS resolution
```

## Remote daemon networking (host-level, not container-level)

```bash
docker -H ssh://user@remote-host ps    # preferred: tunnels over SSH
docker -H tcp://remote-host:2376 ps    # only with TLS client certs — never expose unauthenticated
```

## Removing networks

```bash
docker network rm mynet
docker network prune -f
docker network disconnect mynet myapp && docker network rm mynet   # if containers still attached
```


---
The `overlay` network driver creates a distributed network among multiple Docker daemon hosts. This network sits on top of (overlays) the host-specific networks, allowing containers connected to it to communicate securely when encryption is enabled. Docker transparently handles routing of each packet to and from the correct Docker daemon host and the correct destination container.
#### Overlay Network
```bash
docker network create -d overlay --attachable my-attachable-overlay
```

#### Encrypt traffic on an overlay network
```shell
docker network create \
  --opt encrypted \
  --driver overlay \
  --attachable \
  my-attachable-multi-host-network

```

