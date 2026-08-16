![[Pasted image 20260816191708.png]]

![[Pasted image 20260816191801.png]]

![[Pasted image 20260816192327.png]]

## What taints and tolerations are

Taints and tolerations work together to control **which pods can be scheduled onto which nodes** — but from the opposite direction of `nodeSelector`/`nodeAffinity`.

- **`nodeAffinity`/`nodeSelector`** = the _pod_ says "I want to run on a node with these properties" (pod chooses node).
- **Taints/tolerations** = the _node_ says "don't schedule anything here unless you're specifically allowed" (node repels pods).

**Taint** = applied to a **node**. Repels pods that don't tolerate it. **Toleration** = applied to a **pod**. Allows (but doesn't force) the pod to be scheduled onto a matching tainted node.

Think of a taint as a "keep out" sign, and a toleration as a permission slip that lets a specific pod ignore that sign.

## Tainting a node

```bash
kubectl taint nodes node1 key=value:NoSchedule
```

Format: `key=value:effect`

## The three taint effects

|Effect|Behavior|
|---|---|
|`NoSchedule`|New pods without a matching toleration **won't be scheduled** here. Existing pods already running are unaffected.|
|`PreferNoSchedule`|Soft version — the scheduler **tries to avoid** placing pods here, but will if there's no better option.|
|`NoExecute`|Strongest — new pods without toleration won't schedule, **and existing pods without a toleration get evicted** from the node.|

## Adding a toleration to a pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: rabbit
spec:
  tolerations:
  - key: "key"
    operator: "Equal"
    value: "value"
    effect: "NoSchedule"
  containers:
  - name: app
    image: myapp:latest
```

This means: "I'm allowed on a node tainted with `key=value:NoSchedule`." It does **not** mean the pod _must_ go there — just that it's _permitted_ to.

## `operator: Equal` vs `Exists`

```yaml
tolerations:
- key: "key"
  operator: "Exists"
  effect: "NoSchedule"
```

- `Equal` (default) — key **and** value must match exactly.
- `Exists` — only the key needs to exist on the node (value ignored). Useful for a blanket "tolerate anything with this key" toleration.

You can also tolerate **everything** by omitting `key` entirely with `operator: Exists` — this is how some system-critical DaemonSets (like `kube-proxy`, CNI plugins) tolerate all taints including master/control-plane taints.

## `tolerationSeconds` (for `NoExecute`)

```yaml
tolerations:
- key: "key"
  operator: "Equal"
  value: "value"
  effect: "NoExecute"
  tolerationSeconds: 300
```

Instead of tolerating a `NoExecute` taint forever, the pod stays for a limited window (300s here) after the taint appears, then gets evicted anyway. Useful for graceful handling of node failures — e.g. the built-in `node.kubernetes.io/unreachable` and `node.kubernetes.io/not-ready` taints, which Kubernetes adds automatically when a node goes down, giving pods a grace period before rescheduling elsewhere.

## Common real-world use cases

- **Dedicated nodes** — e.g. GPU nodes tainted so only ML workloads (with the matching toleration) land there, keeping general workloads off expensive hardware.
- **Control-plane nodes** — master nodes are tainted by default (`node-role.kubernetes.io/control-plane:NoSchedule`) so regular workloads don't get scheduled there; only system pods tolerate it.
- **Node problem handling** — Kubernetes auto-taints nodes that are `NotReady` or `unreachable`, triggering eviction of non-tolerating pods (`NoExecute`).
- **Multi-tenancy / isolation** — reserving a pool of nodes for a specific team or environment (e.g. `env=staging:NoSchedule`) so production workloads can't accidentally land there.

## Critical point: taints alone don't guarantee placement

A toleration only **permits** scheduling on a tainted node — it doesn't **force** the pod there. If you want a pod to _only_ run on specific nodes (not just be _allowed_ to), you must combine a toleration with `nodeSelector` or `nodeAffinity`:

```yaml
spec:
  tolerations:
  - key: "gpu"
    operator: "Equal"
    value: "true"
    effect: "NoSchedule"
  nodeSelector:
    gpu: "true"
```

Without the `nodeSelector`, the pod could just as easily land on an untainted node — the toleration only removes the repulsion, it doesn't create attraction.

## Quick commands

```bash
# Add a taint
kubectl taint nodes node1 key=value:NoSchedule

# Remove a taint (note the trailing -)
kubectl taint nodes node1 key=value:NoSchedule-

# View taints on a node
kubectl describe node node1 | grep Taints
```

## Taints/tolerations vs nodeAffinity — quick comparison
![[Pasted image 20260816194454.png]]

They're often used **together**: taint the special nodes to keep everyone else out, then use both a toleration _and_ nodeAffinity/nodeSelector on the pods that should specifically go there — the toleration gets them past the "keep out" sign, and the affinity actually steers them in.

To verify the pod running on the tainted node:
```bash
controlplane ~ ➜  kubectl get pods -A --field-selector spec.nodeName=node01 -o wide
NAMESPACE      NAME                    READY   STATUS    RESTARTS   AGE    IP               NODE     NOMINATED NODE   READINESS GATES
default        bee                     1/1     Running   0          6m3s   172.17.1.2       node01   <none>           <none>
kube-flannel   kube-flannel-ds-hshsb   1/1     Running   0          30m    192.168.34.166   node01   <none>           <none>
kube-system    kube-proxy-26cbl        1/1     Running   0          30m    192.168.34.166   node01   <none>           <none>
```
