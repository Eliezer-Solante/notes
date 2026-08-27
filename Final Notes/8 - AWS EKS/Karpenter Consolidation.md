Consolidation is Karpenter's mechanism for continuously **right-sizing and shrinking** your node fleet after the initial scale-up — it's the "scale down intelligently" half of what Karpenter does. Here's how it actually works.

## The core idea

Karpenter doesn't just add nodes when Pods are Pending — it continuously re-evaluates the whole cluster and asks: _"Could I achieve the same Pod placement with fewer or cheaper nodes?"_ If yes, it takes action.

## Three consolidation triggers

**1. Empty node consolidation** If a node has **zero non-DaemonSet Pods** running on it, Karpenter terminates it almost immediately (after a short delay to avoid flapping). This is the simplest case — no Pods to reschedule, just delete the node.

**2. Underutilized node consolidation (replace)** If Karpenter determines that Pods running across **multiple underutilized nodes** could all fit onto a **single smaller/cheaper instance**, it will:

1. Provision the new, smaller-fitting instance
2. Reschedule the Pods onto it
3. Drain and terminate the old, now-redundant nodes

**3. Underutilized node consolidation (delete only)** If Pods on an underutilized node could simply be **absorbed by existing capacity elsewhere** in the cluster (no new node needed at all), Karpenter just drains that node and deletes it — the Pods get rescheduled onto already-running nodes with spare room.

## The decision loop (simplified)

```
Every evaluation cycle, for each node:
  1. Can this node be deleted with zero disruption? (empty) → delete
  2. Could this node's Pods fit on cheaper/fewer nodes? → simulate the replacement
  3. Is the projected cost lower AND the move safe? → consolidate
  4. Otherwise → leave the node as-is
```

Karpenter is constantly running this simulation in the background, not just reacting to scale-up events.

## What makes a move "safe" (guardrails)

Karpenter won't disrupt a node carelessly — it respects:

- **PodDisruptionBudgets (PDBs)** — won't evict Pods if doing so would violate a PDB
- **`do-not-disrupt` annotation** — Pods explicitly marked with `karpenter.sh/do-not-disrupt: "true"` are never touched (useful for stateful workloads, batch jobs mid-run, etc.)
- **Node/Pod affinity, taints, tolerations** — replacement nodes must still satisfy all the original scheduling constraints
- **Consolidation policy setting on the NodePool** — you can configure whether consolidation is allowed at all, and under what conditions

## Configuring it — `NodePool` spec

```yaml
apiVersion: karpenter.sh/v1
kind: NodePool
metadata:
  name: default
spec:
  disruption:
    consolidationPolicy: WhenEmptyOrUnderutilized   # or WhenEmpty
    consolidateAfter: 30s                             # how long to wait before acting
```

- **`WhenEmpty`** — only consolidate nodes with zero Pods (conservative, minimal disruption)
- **`WhenEmptyOrUnderutilized`** — also actively consolidate underutilized nodes (more aggressive cost savings, but more churn/disruption)
- **`consolidateAfter`** — a cooldown/settle period so Karpenter doesn't thrash on every tiny fluctuation

## Why this matters for cost

This is the main reason teams adopt Karpenter over Cluster Autoscaler — Cluster Autoscaler generally only scales _down_ when a node is **completely empty**, and even then can be conservative. Karpenter actively **repacks and shrinks** the fleet, aggressively chasing the cheapest instance shapes that still satisfy all running workloads — which typically yields meaningfully lower EC2 spend without you managing anything manually.

## Quick example, end to end

```
Before: 3 nodes, each running 1-2 small Pods, mostly idle
              ↓ Karpenter evaluates
Simulation: all Pods could fit on 1 mid-sized node, cheaper overall
              ↓
Action: launch 1 new node → reschedule Pods → drain + delete the 3 old nodes
After: 1 node, fully utilized, lower total cost
```

## Summary

> Karpenter doesn't just scale up on demand — it continuously watches for opportunities to **shrink, merge, or replace** nodes with cheaper/fewer alternatives, as long as doing so doesn't violate any Pod's scheduling constraints or disruption budget. That constant background repacking is what "consolidation" refers to.