## The core distinction

These two mechanisms solve the _same general problem_ — controlling pod-to-node placement — but from **opposite directions**, and they answer different questions:

- **Taints & Tolerations** answer: _"Which pods am I willing to accept?"_ (node's perspective — exclusion-based)
- **Node Affinity** answers: _"Which nodes do I want to run on?"_ (pod's perspective — attraction-based)

This directionality is the single most important thing to internalize — everything else follows from it.

## Side-by-side comparison

||Taints & Tolerations|Node Affinity|
|---|---|---|
|Applied to|Node (taint) + Pod (toleration)|Pod only|
|Direction|Node repels pods|Pod attracts to nodes|
|Default behavior|Pods **excluded** unless they tolerate|Pods can go **anywhere** unless constrained|
|Can it evict running pods?|Yes (`NoExecute`)|No — only affects scheduling, never eviction|
|Soft option?|Yes (`PreferNoSchedule`)|Yes (`preferredDuringScheduling...`)|
|Logic richness|Simple key=value match|Rich: `In`, `NotIn`, `Exists`, `Gt`, `Lt`, AND/OR|
|Guarantees pod lands there?|No — only permits, doesn't attract|Yes, if `required`|
|Typical use|Reserve/protect nodes from general workloads|Steer specific workloads toward specific nodes|

## The critical gotcha: neither one alone gives you what you probably want

This is the thing worth sitting with, because it's the most common source of confusion.

**Taint + toleration alone** → the pod is _permitted_ on the tainted node, but nothing stops the scheduler from placing it on an _untainted_ node instead. A toleration removes a restriction; it creates no pull toward that node.

**Node affinity alone** → the pod is _steered_ toward the labeled node, but nothing stops _other, unrelated_ pods from also landing there — since there's no taint keeping them out.

```
Taint only:        "Keep out, unless you have a permission slip"
                    → doesn't stop the permitted pod from going elsewhere

Affinity only:      "I want to go to that specific building"
                    → doesn't stop other people from also walking in
```

## The standard combined pattern

To get true **dedicated node** behavior — "only these pods go here, and these pods only go here" — you need all three pieces together:

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

|Piece|What it does|Without it|
|---|---|---|
|Taint on node|Repels all other pods|Any pod could land on the GPU node|
|Toleration on pod|Lets this pod past the taint|This pod couldn't schedule there at all|
|Node affinity on pod|Actually pulls this pod toward that node|Pod could just as easily land elsewhere|

Remove any one piece and the isolation breaks down.

## Analogy that ties it together

Picture a VIP section at a venue:

- **Taint** = velvet rope + bouncer — keeps the general crowd out by default.
- **Toleration** = the VIP wristband — lets a specific guest past the bouncer.
- **Node affinity** = the guest's own decision to walk toward the VIP section rather than the main floor.

A wristband (toleration) without deciding to walk over (affinity) means the guest might just stay on the main floor anyway. Deciding to walk toward the VIP section (affinity) without a wristband (toleration) means the bouncer stops them at the rope. You need both to guarantee "this specific guest, and only this guest, ends up there."

## When to use which alone

- **Taints/tolerations alone** — when you just need to _keep the riff-raff out_ and don't care exactly which of your "allowed" pods ends up there (e.g. tainting nodes as `NoExecute` during maintenance so most workloads evacuate, without needing to specify where they should go instead).
- **Node affinity alone** — when you want to _prefer_ certain nodes without needing hard exclusivity (e.g. `preferredDuringScheduling` to softly favor a particular zone for latency reasons, but you're fine if other unrelated pods share that zone too).
- **Both together** — true dedicated/isolated node pools (GPU nodes, high-memory nodes, compliance-restricted nodes).

```
  - effect: NoSchedule
    key: dedicated
    operator: Equal
    value: ml
  - effect: NoExecute
    key: node.kubernetes.io/not-ready
    operator: Exists
    tolerationSeconds: 300
  - effect: NoExecute
    key: node.kubernetes.io/unreachable
    operator: Exists
    tolerationSeconds: 300
```