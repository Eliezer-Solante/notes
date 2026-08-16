Docker Swarm is Docker's native clustering and orchestration tool — it turns a group of Docker hosts into a single virtual host you can deploy to, using the same CLI and Compose-file syntax you already know from plain Docker.

![[Pasted image 20260801231944.png]]

![[Pasted image 20260801232027.png]]

copy and run the `join` command on each worker hosts/node

![[Pasted image 20260801232323.png]]

## Core concepts

**Node** — a machine (physical or VM) running Docker and joined to the swarm. Two roles:

- **Manager node** — maintains cluster state, schedules services, handles the API. Managers use the Raft consensus algorithm to stay in sync, so you typically run an odd number (3 or 5) for fault tolerance.
- **Worker node** — runs containers (tasks) assigned by managers. A manager can also act as a worker unless drained.

**Service** — the declarative definition of what you want running (image, replicas, ports, etc.). Swarm's equivalent of a Kubernetes Deployment.

**Task** — a single running container that's part of a service. Swarm schedules tasks onto nodes.

**Stack** — a group of related services deployed together, typically from a Compose file.

## Setting up a swarm

```bash
# On the first machine, initialize the swarm
docker swarm init --advertise-addr <MANAGER-IP>

# This outputs a join token, use it on other machines:
docker swarm join --token <TOKEN> <MANAGER-IP>:2377

# Check nodes
docker node ls
```

## Deploying a service

```bash
docker service create --name web --replicas 3 -p 80:80 nginx
```

This automatically:

- Spreads 3 replicas across available nodes
- Load-balances traffic across them via Swarm's built-in routing mesh
- Restarts any replica that dies

Check status:

```bash
docker service ls
docker service ps web
```

Scale it:

```bash
docker service scale web=5
```

Rolling update:

```bash
docker service update --image nginx:1.27 web
```

## Deploying a stack (multi-service apps)

Swarm reuses Compose file syntax with a `deploy` key:

```yaml
version: "3.8"
services:
  web:
    image: nginx
    deploy:
      replicas: 3
      restart_policy:
        condition: on-failure
      update_config:
        parallelism: 1
        delay: 10s
    ports:
      - "80:80"
  api:
    image: myregistry.example.com/myapp:v1.0
    deploy:
      replicas: 2
```

Deploy it:

```bash
docker stack deploy -c docker-compose.yml mystack
docker stack services mystack
```

## Networking

Swarm creates an **overlay network** spanning all nodes, so containers on different hosts can talk to each other by service name — Docker's embedded DNS handles service discovery. The **routing mesh** means you can hit any node's published port and get routed to a healthy container, even if it's running on a different node.

## Where Swarm fits vs Kubernetes

**Strengths:**

- Much lower learning curve — if you know `docker run` and Compose, you basically know Swarm
- Built into the Docker Engine, no separate installation
- Simple setup — a working multi-node cluster in minutes
- Good enough for small-to-mid scale deployments

**Weaknesses:**

- Smaller ecosystem — far fewer third-party tools, operators, and integrations than Kubernetes
- Less flexible scheduling and autoscaling (no built-in horizontal pod autoscaler equivalent)
- Weaker support for complex stateful workloads
- Mindshare has shifted heavily to Kubernetes — less active development, fewer job postings/community resources, harder to hire for
- Managed cloud offerings you'll find are almost all Kubernetes (EKS, GKE, AKS); managed Swarm options are rare

## Practical takeaway

Swarm still works and is genuinely simpler to operate for small teams or straightforward deployments. But if you're planning to grow, need advanced scheduling/autoscaling, or want access to the broader tooling ecosystem (Helm, operators, service meshes, etc.), Kubernetes is the more future-proof choice — which is largely why most production orchestration has consolidated around it.

Want me to walk through a full multi-node Swarm setup end to end, or compare specific Swarm vs Kubernetes commands side by side?