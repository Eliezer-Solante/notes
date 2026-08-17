![[Pasted image 20260813175652.png]]



![[Pasted image 20260813175807.png]]


#### NodePort Service
![[Pasted image 20260813175945.png]]
![[Pasted image 20260813180457.png]]


![[Pasted image 20260813180804.png]]

![[Pasted image 20260813180912.png]]

A **NodePort** is one of the four Service `types` in Kubernetes — the mechanism that exposes a Service on a static port on every node's IP address, making it reachable from outside the cluster without needing a cloud load balancer.

## Where NodePort fits among Service types

|Type|Exposes to|Typical use|
|---|---|---|
|ClusterIP (default)|Internal cluster only|Pod-to-pod / service-to-service traffic|
|**NodePort**|External, via `<NodeIP>:<NodePort>`|Dev/test, or bare-metal clusters without a cloud LB|
|LoadBalancer|External, via cloud provider's LB|Production external access on cloud platforms|
|ExternalName|Maps to an external DNS name|Pointing to a service outside the cluster|

NodePort actually builds **on top of** ClusterIP — every NodePort Service also gets a ClusterIP automatically, so it's reachable both ways.

## How NodePort works

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-app-svc
spec:
  type: NodePort
  selector:
    app: my-app
  ports:
    - port: 80          # ClusterIP port (internal)
      targetPort: 8080   # container port the pod is listening on
      nodePort: 30080    # external port opened on every node (30000-32767 range)
```

Once applied, **every node in the cluster** opens port `30080`, regardless of whether that node is actually running one of the matching pods. Traffic hitting any node on that port gets routed (via `kube-proxy`) to one of the healthy pods matching the `selector` — even if that pod lives on a different node entirely.

```
                  external client
                        │
              hits any node:30080
                        │
        ┌───────────────┴───────────────┐
        │                                │
     Node A                          Node B
   (no pod here,                  (pod running)
   but still routes) ──────────────────┘
```

## Common commands

```bash
# apply
kubectl apply -f service.yaml

# check assigned NodePort (auto-assigned if you omit it)
kubectl get svc my-app-svc
# NAME          TYPE       CLUSTER-IP     PORT(S)
# my-app-svc    NodePort   10.96.10.20    80:30080/TCP

# get node IPs to actually hit it
kubectl get nodes -o wide

# then from outside the cluster:
curl http://<any-node-ip>:30080

# describe (shows endpoints — which pods it's actually routing to)
kubectl describe svc my-app-svc

# delete
kubectl delete svc my-app-svc
```

## Key things worth knowing

- **Port range is fixed**: `30000–32767` by default (configurable at the cluster level). You can specify a `nodePort` manually within that range, or let Kubernetes auto-assign one.
- **Not meant for production external traffic** — exposing raw node IPs and high ports to the internet is clunky and insecure at scale. In cloud environments, `LoadBalancer` (which usually provisions a NodePort _and_ a cloud LB in front of it) or an `Ingress` are the standard production paths.
- **`endpoints`** — a Service doesn't route to pods directly; it maintains an `Endpoints` object listing the actual pod IPs matching its `selector`, updated live as pods come and go. `kubectl get endpoints my-app-svc` shows this.
- **Ties back to what you asked earlier**: the same DNS resolution (`my-app-svc.dev.svc.cluster.local`) works for internal calls to a NodePort service too — NodePort just _adds_ external reachability, it doesn't remove the internal ClusterIP path.
