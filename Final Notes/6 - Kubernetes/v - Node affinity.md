![[Pasted image 20260816203234.png]]
add `or` or another values for the pod to be placed in 
![[Pasted image 20260816203343.png]]



## What node affinity is

`nodeAffinity` is the more expressive, modern successor to `nodeSelector`. Same goal — constrain which nodes a pod can run on based on labels — but with richer logic: OR conditions, "not in" exclusions, and crucially, the ability to express **soft preferences** instead of only hard requirements.

## The two types

|Type|Behavior|Analogy|
|---|---|---|
|`requiredDuringSchedulingIgnoredDuringExecution`|**Hard rule.** Pod won't schedule unless satisfied.|Same as `nodeSelector`, just richer syntax|
|`preferredDuringSchedulingIgnoredDuringExecution`|**Soft rule.** Scheduler tries to honor it, but will place the pod elsewhere if it can't.|"Nice to have"|

The `IgnoredDuringExecution` suffix on both means: once the pod is running, if the node's labels change and no longer match, **the pod is not evicted**. Affinity is only checked at scheduling time, not enforced continuously. (There's talked-about future `RequiredDuringExecution` support but it's not standard yet — don't rely on it.)

## Hard requirement example

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: rabbit
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: disktype
            operator: In
            values:
            - ssd
            - nvme
  containers:
  - name: app
    image: myapp:latest
```

This is the OR logic `nodeSelector` can't do: "node must have `disktype=ssd` OR `disktype=nvme`."

## Soft preference example

```yaml
spec:
  affinity:
    nodeAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 80
        preference:
          matchExpressions:
          - key: zone
            operator: In
            values:
            - us-east-1a
      - weight: 20
        preference:
          matchExpressions:
          - key: zone
            operator: In
            values:
            - us-east-1b
```

- `weight` (1–100) — higher weight = stronger preference. The scheduler sums weights across all matching preferred terms to score candidate nodes, then picks the highest-scoring one.
- If **no** node matches any preference, the pod still schedules — just wherever else fits. This is the key difference from `required`.

## Operators available in `matchExpressions`

|Operator|Meaning|
|---|---|
|`In`|Label value is in the given list|
|`NotIn`|Label value is NOT in the given list|
|`Exists`|Label key exists (value ignored)|
|`DoesNotExist`|Label key does not exist|
|`Gt`|Label value (numeric) greater than given value|
|`Lt`|Label value (numeric) less than given value|

`Gt`/`Lt` are useful for things like `capacity > 100` if you label nodes with numeric values.

## Combining required + preferred

You can use both simultaneously — required narrows the candidate pool, preferred ranks within it:

```yaml
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: disktype
            operator: In
            values: ["ssd"]
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        preference:
          matchExpressions:
          - key: zone
            operator: In
            values: ["us-east-1a"]
```

"Must be SSD. Prefer zone `us-east-1a` if possible, but any SSD node is acceptable."

## Multiple `nodeSelectorTerms` vs multiple `matchExpressions`

This trips people up — they behave differently:

- Multiple **`nodeSelectorTerms`** (top-level array) = **OR** between terms.
- Multiple **`matchExpressions`** within a single term = **AND** within that term.

```yaml
nodeSelectorTerms:
- matchExpressions:              # Term 1
  - key: disktype
    operator: In
    values: ["ssd"]
  - key: zone
    operator: In
    values: ["us-east-1a"]
- matchExpressions:               # Term 2
  - key: disktype
    operator: In
    values: ["nvme"]
```

Reads as: `(disktype=ssd AND zone=us-east-1a) OR (disktype=nvme)`.

## nodeAffinity vs nodeSelector vs taints/tolerations
![[Pasted image 20260816210107.png]]

## Where it fits with everything else we've covered

The full picture for "make sure only the right pods land on the right nodes":

- **Taint the node** → repel everything by default.
- **Toleration on the pod** → lets the specific pod get past the repel.
- **nodeAffinity/nodeSelector on the pod** → actually steers the pod there, rather than leaving it to chance.

```yaml
spec:
  tolerations:
  - key: "gpu"
    operator: "Equal"
    value: "true"
    effect: "NoSchedule"
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: gpu
            operator: In
            values: ["true"]
```

## Related: podAffinity / podAntiAffinity (worth knowing exists)

Same mechanism, different target — instead of matching _node_ labels, these match based on **labels of other pods already running**:

- `podAffinity` — "schedule me near pods with label X" (e.g. co-locate a cache with the app that uses it, reducing network hops).
- `podAntiAffinity` — "don't schedule me on the same node as pods with label X" (e.g. spread replicas of the same Deployment across nodes for high availability).

Not the same as nodeAffinity, but same syntax style and often used alongside it — worth a separate discussion if you want to go there next.

## Quick verification

```bash
kubectl describe pod rabbit | grep -A 15 Affinity
kubectl get pod rabbit -o jsonpath='{.spec.affinity}'
```

If a pod with `required` affinity is stuck `Pending`:

```bash
kubectl describe pod rabbit
# Look for: "0/3 nodes are available: 3 node(s) didn't match Pod's node affinity"
```