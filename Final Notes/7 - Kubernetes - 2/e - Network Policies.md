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
- Apply defaults cluster-wide via GitOps/Kyverno/OPA policies rather than manually per namespace.

INGRESS ANALOGY:

**Think of your Kubernetes cluster as a big office building with many different departments (Services) inside — HR, Sales, Support, Billing, etc.**

**Without Ingress (NodePort approach):**  
Imagine every department has its **own separate street-facing door**, each with a different, oddly-numbered address (like `30011`, `30012`, `30013`...). Visitors have to already know the exact door number for the department they want. It's clunky, hard to remember, and doesn't scale — imagine memorizing 50 different door numbers for 50 departments.

**Without Ingress (LoadBalancer approach):**  
Now imagine instead each department builds its **own private entrance with its own dedicated security guard/reception desk** (a cloud load balancer). It works, but now you're paying for and maintaining 50 separate front desks — expensive and wasteful, since most of them are doing the same basic job of "let people in."

**With Ingress:**  
The building has **one main lobby with a single receptionist** (the Ingress Controller) at the main entrance (port 80/443). Everyone walks in through the same front door. The receptionist looks at _who you're asking for_ — either the name on the building directory (**hostname-based routing**, e.g. `billing.company.com` vs `hr.company.com`) or _which floor/room you asked for_ (**path-based routing**, e.g. `/billing` vs `/hr`) — and directs you to the right department internally.

You, the visitor, only ever need to know **one address**: the front door. The receptionist (Ingress) handles all the internal routing logic, so the company doesn't need a separate expensive entrance for every department.

That's exactly what Ingress does for a cluster: **one entry point, smart routing rules, to many backend Services** — instead of many entry points (NodePort) or many expensive dedicated ones (LoadBalancer per service).


This is a genuinely common point of confusion because Kubernetes reuses the same word "Ingress" for two completely different things. Let's separate them clearly.

**1. Ingress (the standalone API object) — "Ingress Networking"**

- **API:** `kind: Ingress` (`networking.k8s.io/v1`)
- **Purpose:** Routes **external HTTP/HTTPS traffic** into the cluster, to specific Services, based on **hostname or URL path** rules.
- **Layer:** Operates at **Layer 7** (application layer — understands HTTP, hosts, paths, TLS/SSL termination).
- **Requires:** An **Ingress Controller** (nginx-ingress, Traefik, etc.) actually running to fulfill the rules — the Ingress object alone does nothing without a controller.
- **Example use case:** "Send requests for `api.myapp.com` to the `api-service`, and requests for `web.myapp.com` to the `web-service`."
- This is the receptionist/lobby analogy from earlier — it's about **how outside visitors get routed to the right internal department**.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp-ingress
spec:
  rules:
  - host: api.myapp.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 80
```

---

**2. Ingress (as a rule type inside NetworkPolicy)**

- **API:** Part of `kind: NetworkPolicy` (`networking.k8s.io/v1`), under `policyTypes: [Ingress]`
- **Purpose:** Controls which **pods/traffic sources are allowed to send traffic INTO a given pod** — pure network-level access control.
- **Layer:** Operates at **Layer 3/4** (IP addresses, namespaces, pod labels, ports/protocols — no awareness of HTTP, hostnames, or paths).
- **Requires:** A **CNI plugin that supports NetworkPolicy enforcement** (e.g., Calico, Cilium) — plain Flannel, for example, won't enforce these rules at all.
- **Example use case:** "Only pods labeled `app: frontend` are allowed to send traffic to this pod on port 5432 — block everyone else, including other namespaces."

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: restrict-ingress
spec:
  podSelector:
    matchLabels:
      app: database
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: frontend
    ports:
    - protocol: TCP
      port: 5432
```

---

**Side-by-side summary:**

||Ingress (object)|Ingress (in NetworkPolicy)|
|---|---|---|
|Direction concern|External traffic **into the cluster**|Traffic **into a specific pod** (from anywhere, internal or external)|
|Layer|L7 (HTTP-aware)|L3/L4 (IP/port-aware only)|
|Routes based on|Hostname, URL path|Pod labels, namespace, IP CIDR, port|
|Needs|Ingress Controller|CNI with policy support (Calico, Cilium, etc.)|
|Analogy|The building's front-door receptionist deciding **which department** you go to|A department's own **door lock** deciding **who's allowed to enter that specific room**, regardless of how they got into the building|

**The key distinction to hold onto:** the standalone **Ingress object** is about _smart HTTP routing from outside the cluster to a Service_. The **Ingress rule in NetworkPolicy** is about _firewall-style access control for traffic reaching a pod_, and has nothing to do with HTTP routing, hostnames, or paths at all. They just happen to share the same English word because both describe traffic moving "inward" — from two very different vantage points.