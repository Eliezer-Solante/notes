Before you can apply for `nodeSelector`, label the nodes first
![[Pasted image 20260816195147.png]]

labeling the node 
![[Pasted image 20260816195300.png]]


so, when creating the pod, it is then place under that particular labeled node, in this case `size: Large`
![[Pasted image 20260816195419.png]]

Limitations
![[Pasted image 20260816195602.png]]



### What nodeSelector is

`nodeSelector` is the simplest way to constrain a pod to run only on nodes that have specific **labels**. It's the opposite direction from taints/tolerations:

- **Taints/tolerations** = node repels pods unless permitted (exclusion-based)
- **nodeSelector** = pod requests a node with certain properties (attraction-based, but hard requirement)

Think of it as: "Only schedule me on a node labeled `disktype=ssd`." If no such node exists, the pod stays `Pending` — it never falls back to a random node.

### Step 1: Label the node

bash

```bash
kubectl label nodes node1 disktype=ssd
```

Check labels on a node:

bash

```bash
kubectl get nodes --show-labels
```

### Step 2: Add `nodeSelector` to the pod spec

yaml

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: rabbit
spec:
  nodeSelector:
    disktype: ssd
  containers:
  - name: app
    image: myapp:latest
```

The pod will only be scheduled onto a node that has the label `disktype=ssd`. If it's part of a Deployment, the `nodeSelector` goes under `spec.template.spec` instead.