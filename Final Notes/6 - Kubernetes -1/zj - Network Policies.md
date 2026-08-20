Basically, communication restrictions of who can communicate with each other
**NetworkPolicy = an allow-list that says "this pod may only send/receive traffic to/from these specific pods, on these specific ports."** Everything not explicitly allowed gets dropped, once a policy touches that pod for that direction.

A few nuances worth pinning down now that you've got the core idea, since they follow directly from "it's a restriction mechanism":

### It's allow-only, never deny-only

You can't write a rule that says "block pod X." You can only write rules that say "allow pod Y." So to block something, you either:

- Don't include it in any allow rule (the default-deny-once-selected behavior does the blocking for you), or
- Restructure your allow rules so the unwanted source/destination simply isn't covered

There's no explicit `deny` verb in the API — restriction is achieved entirely through _omission_ from allow rules.

### It restricts by identity (labels), not by network topology

This is the other half of "who can communicate" — the "who" is defined by **pod labels / namespace labels / IP blocks**, not by things like node placement, subnet, or physical location. Two pods could be on completely different nodes, different racks, different AZs — if their labels match an allow rule, they communicate. Two pods could be on the exact same node — if labels don't match, they're blocked. Identity-based, not location-based.

### It's directional and independent

"Who can talk to whom" has two separate halves that both need answering, per pod:

- **Ingress** — who's allowed to initiate a connection _to_ this pod
- **Egress** — who this pod is allowed to initiate a connection _out to_

They're tracked separately. A pod can have wide-open egress but locked-down ingress, or vice versa, or both restricted, or neither.

### And restriction happens per-pod, cumulative across policies

If pod P is selected by three different NetworkPolicy objects, the effective rule set is the **union (OR)** of all three — not an intersection. More policies targeting the same pod = more allowed communication, never less. (You can't "further restrict" an already-restricted pod by adding another policy — you can only widen it.)

![[Pasted image 20260820094712.png]]
![[Pasted image 20260820094733.png]]
![[Pasted image 20260820094754.png]]
![[Pasted image 20260820094841.png]]
This diagram is showing a classic **3-tier application traffic flow**, and how each hop involves both an ingress and an egress side (from the perspective of whichever pod/service is receiving or sending).

## The three tiers

|Tier|Icon|Represents|
|---|---|---|
|Top|🌐 Globe|Web/Frontend service (public-facing)|
|Middle|API|Backend API service|
|Bottom|🗄️ Cylinder|Database|

## Walking through the numbered flow

**1 → Ingress :80** — Traffic arrives at the **Web/Frontend** tier on port 80 (standard HTTP). This is the entry point for external users hitting your app.

**2 → Egress :5000** — The Frontend, to fulfill that request, makes an **outbound** call to the API tier on port 5000. From the frontend's perspective this is egress (it's leaving the frontend pod).

**3 → Ingress :5000** — That same call **arrives** at the API tier on port 5000. This is the API's ingress — same connection as step 2, just viewed from the receiving side.

**4 → Egress :3306** — The API, to look up/store data, makes an **outbound** call to the Database on port 3306 (MySQL's default port). This is egress from the API pod.

**5 → Ingress :3306** — That connection **arrives** at the Database on port 3306. This is the database's ingress.

## The key concept this illustrates

Every single connection in the chain has **two sides**:

- **Egress** from the caller
- **Ingress** to the callee

This matters a lot for Kubernetes NetworkPolicies specifically, because you have to allow _both_ directions or traffic breaks:

```yaml
# Frontend pod needs:
egress: allow → API pods, port 5000

# API pod needs:
ingress: allow ← Frontend pods, port 5000
egress:  allow → DB pods, port 3306

# DB pod needs:
ingress: allow ← API pods, port 3306
```

If you only write the egress rule on the Frontend but forget the matching ingress rule on the API, the connection still fails — NetworkPolicy requires **both ends to permit the traffic** (assuming both pods are governed by a policy in that direction). This diagram is a nice visual reminder that "one connection = two policy directions to configure."


![[Pasted image 20260820095415.png]]



![[Pasted image 20260820101029.png]]
![[Pasted image 20260820101239.png]]
![[Pasted image 20260820101313.png]]


![[Pasted image 20260820101340.png]]
---
# Kubernetes NetworkPolicies: Ingress and Egress

## The Basics

By default, Kubernetes networking is **flat and open** — every pod can talk to every other pod in the cluster, regardless of namespace. A `NetworkPolicy` resource lets you restrict that traffic at the pod level, acting like a firewall implemented by your CNI plugin (Calico, Cilium, Weave, etc. — the API does nothing without a compliant network plugin).

Two directions of traffic control:

- **Ingress** — controls incoming connections _to_ a pod
- **Egress** — controls outgoing connections _from_ a pod

## Key Concept: Default Deny vs. Default Allow

This trips people up constantly:

- If **no** NetworkPolicy selects a pod, all traffic (ingress and egress) is allowed to/from it.
- The moment **any** NetworkPolicy selects a pod for a given direction (ingress or egress), that direction becomes **default deny** for that pod, except for what the policy explicitly allows.
- Ingress and egress are independent — a policy that only defines `ingress` rules leaves egress completely open (and vice versa).

This means policies are **additive** (allow-list only). You can't write a "deny" rule — you achieve deny by _not_ allowing something, or by combining policies carefully.

## Structure of a NetworkPolicy

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: api-policy
  namespace: backend
spec:
  podSelector:
    matchLabels:
      app: api
  policyTypes:
    - Ingress
    - Egress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: frontend
        - namespaceSelector:
            matchLabels:
              team: payments
      ports:
        - protocol: TCP
          port: 8080
  egress:
    - to:
        - podSelector:
            matchLabels:
              app: database
      ports:
        - protocol: TCP
          port: 5432
    - to:
        - ipBlock:
            cidr: 0.0.0.0/0
            except:
              - 169.254.169.254/32   # block cloud metadata endpoint
      ports:
        - protocol: TCP
          port: 443
```

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: internal-policy
  namespace: default
spec:
  podSelector:
    matchLabels:
      name: internal
  policyTypes:
    - Egress
  egress:
    - to:
        - podSelector:
            matchLabels:
              name: payroll
      ports:
        - protocol: TCP
          port: 8080
    - to:
        - podSelector:
            matchLabels:
              name: mysql
      ports:
        - protocol: TCP
          port: 3306
          
    #ensure that you allow egress traffic to DNS ports TCP and UDP (port 53) to enable DNS resolution from the internal pod.
    - ports:
      - port: 53    
        protocol: UDP
      - port: 53
        protocol: TCP                     
```
Breaking this down:

- **`podSelector`** — which pods this policy applies to (empty selector = all pods in the namespace)
- **`policyTypes`** — declares whether this policy governs Ingress, Egress, or both. Important gotcha: if you define an `egress` block but forget to list `Egress` in `policyTypes`, it's ignored.
- **`from` / `to`** — sources (ingress) or destinations (egress) allowed. Can be:
    - `podSelector` (pods matching labels, same namespace unless combined)
    - `namespaceSelector` (pods in namespaces matching labels)
    - `podSelector` + `namespaceSelector` together (AND logic — pods with X label _in_ namespaces with Y label)
    - `ipBlock` (CIDR ranges, for traffic outside the cluster)
- **`ports`** — restrict by protocol/port; omitting `ports` allows all ports to the allowed peers

## Common Patterns

**Default deny-all ingress (per namespace):**

```yaml
spec:
  podSelector: {}
  policyTypes:
    - Ingress
```

No `ingress` rules defined = nothing is allowed in. Good baseline — then add specific allow policies on top.

**Default deny-all egress:**

```yaml
spec:
  podSelector: {}
  policyTypes:
    - Egress
```

This also blocks DNS lookups unless you explicitly allow egress to kube-dns/CoreDNS on port 53 — a very common gotcha that breaks apps mysteriously.

**Allow only intra-namespace traffic:**

```yaml
spec:
  podSelector: {}
  ingress:
    - from:
        - podSelector: {}
  policyTypes:
    - Ingress
```

## Things That Catch People Out

1. **Multiple policies on the same pod are additive (OR'd), not AND'd.** If Policy A allows port 80 from frontend and Policy B allows port 443 from anywhere, the pod gets both — policies don't restrict each other further.
2. **Empty `podSelector: {}`** matches _all pods_ in the namespace, not "no pods."
3. **`namespaceSelector: {}`** (empty) matches all namespaces, including the pod's own.
4. **NetworkPolicies are namespace-scoped** — a policy in namespace A can still allow traffic _from_ namespace B via `namespaceSelector`, but the policy resource itself lives in the namespace of the pods it selects.
5. **DNS breaks first** when you start locking down egress — always allow UDP/TCP 53 to your DNS pods early.
6. **Not all CNIs enforce egress equally well** — some older plugins only supported ingress. Worth checking your CNI's NetworkPolicy support matrix before relying on egress rules for security.
7. **NetworkPolicy ≠ encryption.** It controls reachability, not confidentiality. For mTLS/encryption you need a service mesh (Istio, Linkerd) or CNI-level encryption (e.g., Cilium with WireGuard).

## Typical Rollout Strategy

- **Options** for how teams typically introduce policies, in order of safety:
- Start with `podSelector: {}` deny-all in a non-prod namespace and watch what breaks (via CNI flow logs, e.g., Cilium Hubble or Calico flow logs)
- Layer in explicit allows per service pair based on actual observed traffic
- Extend to egress once ingress is stable, remembering DNS and any external API dependencies (package registries, cloud metadata, third-party APIs)
- Apply defaults cluster-wide via GitOps/Kyverno/OPA policies rather than manually per namespace

