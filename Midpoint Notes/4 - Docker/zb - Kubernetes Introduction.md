Kubernetes (K8s) is the industry-standard container orchestration platform — it automates deploying, scaling, networking, and healing containerized applications across a cluster of machines.

![[Pasted image 20260801232437.png]]![[Pasted image 20260801232532.png]]




![[Pasted image 20260801232802.png]]

![[Pasted image 20260801232838.png]]
This diagram gives a good visual breakdown of the four core components that make up the Kubernetes control plane — the "brain" of the cluster that makes global decisions and reacts to cluster events (like a pod dying).

## api-server

The front door to Kubernetes. Every request — whether it comes from `kubectl`, a CI/CD pipeline, a controller, or another component — goes through the API server first. It's the only component that talks directly to `etcd`.

The card's sub-items (users, mgmt devices, command-line) represent the different clients hitting it:

- **users** → things like `kubectl apply`, dashboards, CI pipelines
- **mgmt devices** → other cluster components (scheduler, controllers) reading/writing state
- **command-line** → direct CLI/API calls

It validates and authenticates every request, then reads or writes cluster state via etcd.

## etcd

A distributed, consistent key-value store — the single source of truth for the entire cluster's state. Everything you see in the diagram (pods, nodes, services, configs, secrets) is ultimately just a record in etcd.

- Nothing else talks to etcd directly except the api-server — this keeps a single, consistent access path
- Because it's the source of truth, etcd is the most critical piece to back up; losing it means losing your cluster's state
- Typically run as a clustered/replicated store (odd number of nodes) for fault tolerance, similar to how Swarm managers use Raft

## scheduler

Watches the api-server for newly created pods that don't yet have a node assigned, then decides which node they should run on — based on resource requests, constraints (affinity/anti-affinity, taints/tolerations), and available capacity.

Important nuance: the scheduler only _decides placement_ — it doesn't actually start the container. It writes that decision back through the api-server, and the kubelet on the chosen node picks it up and runs it.

## controllers

The reconciliation engine. Each controller runs a loop that watches actual state vs. desired state and works to close the gap. Examples:

- **Deployment controller** — ensures the right number of pod replicas exist
- **Node controller** — notices when a node goes unreachable and marks it accordingly
- **Job controller** — tracks batch job completion

This is what gives Kubernetes its self-healing property: if a pod dies, a controller notices the actual state no longer matches desired state and creates a replacement.

## How these tie together in a request flow

1. You run `kubectl apply -f deployment.yaml`
2. Request hits the **api-server**, gets validated, written to **etcd**
3. **Controller** (Deployment controller) notices new desired state, creates pod objects (also via api-server → etcd)
4. **Scheduler** notices unscheduled pods, assigns them to nodes (again via api-server → etcd)
5. The **kubelet** on each assigned node (not shown here — that's part of the worker node, not control plane) picks up its assignment and actually starts the container

Every arrow in that flow passes through the api-server — it's the hub everything else radiates from, which is exactly what the diagram emphasizes by placing it first and giving it the most sub-items.

Want me to do a companion diagram of the worker node side (kubelet, kube-proxy, container runtime) to complete the picture?


![[Pasted image 20260801233009.png]]

![[Pasted image 20260801233053.png]]



![[Pasted image 20260801233251.png]]

## Core building blocks

- **Cluster** — a set of machines: **control plane** (manages state, scheduling) + **worker nodes** (run workloads)
- **Pod** — smallest deployable unit; one or more containers that share network/storage
- **Deployment** — manages replica pods, handles rolling updates and rollbacks
- **Service** — stable network endpoint/load balancer for a set of pods (pods are ephemeral and get new IPs constantly, Services solve that)
- **Ingress** — routes external HTTP(S) traffic into the cluster
- **ConfigMap / Secret** — inject configuration and sensitive data
- **Namespace** — logical partitioning of a cluster (e.g., dev/staging/prod)

## Basic example

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
      - name: myapp
        image: myregistry.example.com/myapp:v1.0
        ports:
        - containerPort: 8080
```

```bash
kubectl apply -f deployment.yaml
kubectl get pods
kubectl scale deployment myapp --replicas=5
```

## Why it dominates

- Self-healing (auto-restarts, reschedules failed pods)
- Horizontal Pod Autoscaling based on CPU/memory/custom metrics
- Huge ecosystem: Helm (packaging), operators, service meshes (Istio/Linkerd), monitoring (Prometheus/Grafana)
- Available everywhere: self-hosted, or managed as EKS (AWS), GKE (Google), AKS (Azure)

## Tradeoff

Much steeper learning curve than Docker Swarm and more operational complexity — but it's the safest long-term bet given ecosystem size, cloud support, and hiring pool.

Want me to go deeper on any piece — networking model, Helm, autoscaling, or setting up a local cluster (minikube/kind) to try it out?