![[Pasted image 20260813181955.png]]
![[Pasted image 20260813182005.png]]

**ClusterIP** is the default Service type — it gives a Service a stable, internal-only IP address that other things _inside_ the cluster can use to reach a set of pods, without ever knowing or caring about individual pod IPs (which change every time a pod restarts).

## Why it exists

Pods are ephemeral — a Deployment can kill and recreate them at any time, and each new pod gets a new IP. If your frontend pod tried to talk directly to a backend pod's IP, that connection would break the moment the backend pod restarted. A ClusterIP Service solves this by giving you one **stable** virtual IP that always routes to whichever pods currently match its `selector`.

## Example YAML

```yaml
apiVersion: v1
kind: Service
metadata:
  name: db-service
spec:
  type: ClusterIP   # this is the default — can be omitted entirely
  selector:
    app: mysql
  ports:
    - port: 3306         # port the Service exposes internally
      targetPort: 3306    # port the container is actually listening on
```

## How the routing actually works

The Service itself doesn't run anything — it's a virtual IP maintained by `kube-proxy` on every node. Under the hood:

1. Kubernetes watches for pods matching `selector: app: mysql`
2. It populates an `Endpoints` object with their live pod IPs
3. `kube-proxy` sets up iptables (or IPVS) rules so traffic sent to the Service's ClusterIP gets load-balanced across those pod IPs
4. As pods come and go, the Endpoints list updates automatically — the ClusterIP itself never changes

```
     web-pod  ──►  db-service (ClusterIP: 10.96.10.20)  ──►  mysql-pod-a
                                                          ──►  mysql-pod-b
```

## Common commands

```bash
kubectl apply -f service.yaml

# see the assigned ClusterIP
kubectl get svc db-service
# NAME         TYPE        CLUSTER-IP     PORT(S)
# db-service   ClusterIP   10.96.10.20    3306/TCP

# see which pods it's actually routing to
kubectl get endpoints db-service

# describe (selector, ports, endpoints all in one view)
kubectl describe svc db-service

# delete
kubectl delete svc db-service
```

## How this connects to everything we've covered

- **DNS**: this is exactly the `mysql.connect("db-service")` from your image — the DNS name resolves to this ClusterIP, which is why you never hardcode pod IPs.
- **Namespaces**: ClusterIP is only reachable _inside_ the cluster, and by default only easily by short name _within the same namespace_ — cross-namespace calls need the FQDN (`db-service.dev.svc.cluster.local`) we discussed.
- **NodePort**: builds directly on top of ClusterIP — a NodePort Service still has a ClusterIP for internal traffic, it just adds an extra external-facing port on top.

## The one thing to remember

**ClusterIP = internal only.** Nothing outside the cluster (no external client, no `curl` from your laptop) can reach it — that's precisely the gap NodePort and LoadBalancer exist to fill.


CREATING A SERVICE USING `expose` 

### Breaking down the command



```bash
kubectl expose pod redis --port 6379 --name redis-service --type ClusterIP
```

|Part|Meaning|
|---|---|
|`expose pod redis`|Look at the existing pod named `redis`, and use its labels to build a `selector` automatically|
|`--port 6379`|The port the Service will expose (both `port` and `targetPort` default to this unless you specify otherwise)|
|`--name redis-service`|Name of the new Service object|
|`--type ClusterIP`|The Service type (ClusterIP is the default anyway, so this is optional here)|

### The YAML it's equivalent to

```yaml
apiVersion: v1
kind: Service
metadata:
  name: redis-service
spec:
  type: ClusterIP
  selector:
    app: redis          # pulled directly from the redis pod's own labels
  ports:
    - port: 6379
      targetPort: 6379
```

The key trick: `kubectl expose` reads the **labels already on the `redis` pod** and copies them into the new Service's `selector` field automatically. If the `redis` pod has `labels: { app: redis }`, that's exactly what gets used to match traffic to it — you don't type the selector yourself.