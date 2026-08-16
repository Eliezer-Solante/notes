
![[Pasted image 20260815215151.png]]

![[Pasted image 20260815215517.png]]

![[Pasted image 20260815215918.png]]


![[Pasted image 20260815220342.png]]

CPU
![[Pasted image 20260815220549.png]]

Memory
![[Pasted image 20260815220623.png]]

Resource Quota
![[Pasted image 20260815220657.png]]

## LimitRange vs ResourceQuota — quick comparison

![[Pasted image 20260815221227.png]]

Both are **namespace-scoped admission checks**. A pod that violates either is rejected outright at creation — nothing partially applies.

## LimitRange examples

**CPU only**

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: cpu-limits
  namespace: my-namespace
spec:
  limits:
  - type: Container
    default:
      cpu: "500m"
    defaultRequest:
      cpu: "250m"
    min:
      cpu: "100m"
    max:
      cpu: "2"
    maxLimitRequestRatio:
      cpu: "4"
```

**Memory only**

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: memory-limits
  namespace: my-namespace
spec:
  limits:
  - type: Container
    default:
      memory: "512Mi"
    defaultRequest:
      memory: "256Mi"
    min:
      memory: "64Mi"
    max:
      memory: "2Gi"
    maxLimitRequestRatio:
      memory: "4"
```

**Combined (CPU + Memory)**

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: cpu-mem-limits
  namespace: my-namespace
spec:
  limits:
  - type: Container
    default:
      cpu: "500m"
      memory: "512Mi"
    defaultRequest:
      cpu: "250m"
      memory: "256Mi"
    min:
      cpu: "100m"
      memory: "64Mi"
    max:
      cpu: "2"
      memory: "2Gi"
    maxLimitRequestRatio:
      cpu: "4"
      memory: "4"
```

## ResourceQuota example

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: team-quota
  namespace: my-namespace
spec:
  hard:
    requests.cpu: "10"
    requests.memory: 20Gi
    limits.cpu: "20"
    limits.memory: 40Gi
    pods: "20"
    persistentvolumeclaims: "10"
    requests.storage: 100Gi
```

## Order of evaluation

1. Pod submitted →
2. **LimitRange** applies first: injects missing defaults, validates min/max/ratio →
3. **ResourceQuota** applies next: checks if the pod pushes the namespace over its `hard` totals →
4. Fail either step → pod rejected, nothing created.

## Do you need to "inject" these into a pod?

**No.** Neither object is referenced or attached in your pod spec. Once you `kubectl apply` a `LimitRange` or `ResourceQuota` into a namespace, it's automatically enforced by the API server's admission controllers for **every** pod created in that namespace afterward — no annotation, label, or field needed on the pod.

Two things worth knowing:

- **Not retroactive** — pods that already existed before you created the LimitRange/ResourceQuota are unaffected. Only new pod creations (or pods recreated via rollout/restart) get evaluated against them.
- **If your pod already sets its own requests/limits**, LimitRange won't override them — it only fills in what's _missing_, then validates everything (including your explicit values) against min/max/ratio. ResourceQuota just tallies whatever ends up on the pod, whether it came from you or was defaulted by LimitRange.

So the deployment flow is simply: create the namespace → apply LimitRange → apply ResourceQuota → deploy pods normally. The enforcement is automatic from that point on.

## Requests vs Limits

**Requests** = what a container is guaranteed to get. The scheduler uses this number to decide which node to place the pod on — it will only schedule a pod onto a node that has at least this much unallocated capacity. Think of it as a reservation.

**Limits** = the hard ceiling a container is not allowed to exceed. Kubernetes enforces this at runtime via the container runtime/kernel (cgroups), regardless of what else is happening on the node.

```yaml
resources:
  requests:
    cpu: "250m"
    memory: "256Mi"
  limits:
    cpu: "500m"
    memory: "512Mi"
```

## Why CPU and Memory behave differently when limits are hit

This is the important part, because a lot of people assume they behave the same way — they don't.

### CPU is compressible

CPU is a "compressible" resource — a process can be paused and resumed without losing anything. So when a container tries to use more CPU than its limit:

- The kernel **throttles** it. The container doesn't get killed — it just gets less CPU time than it wants, using cgroup CFS quotas.
- Throttling can cause latency spikes and slow performance, but the container keeps running.
- If you don't set a CPU limit at all, the container can burst and use spare CPU on the node (as long as it doesn't starve others below their requests).

**Request behavior for CPU:** guarantees a minimum share of CPU time. If the node is idle, a container can use more than its request; it only gets squeezed back down toward its request when the node is under contention.

### Memory is incompressible

Memory is "incompressible" — you can't pause and rewind memory usage, there's no "give it back later." So when a container tries to use more memory than its limit:

- The kernel **kills** the container (OOMKilled). No throttling option exists for memory the way it does for CPU.
- The pod usually gets restarted, but if it keeps hitting the limit, you'll see `CrashLoopBackOff`.

**Request behavior for memory:** this is what's used for scheduling and also matters during **node pressure**. If a node runs low on memory, kubelet evicts pods, and it prioritizes evicting pods whose memory usage most exceeds their _request_ — pods that are well within their requests are the last to go.

## Quick comparison table

||CPU|Memory|
|---|---|---|
|Exceeding request|Allowed if spare capacity exists|Allowed if spare capacity exists|
|Exceeding limit|Throttled (slowed down, not killed)|Killed (OOMKilled)|
|Resource type|Compressible|Incompressible|
|Node pressure behavior|Throttled fairly across pods|Kubelet evicts pods exceeding requests first|

## A few practical implications

- **Setting CPU limits too low** is a common self-inflicted wound — people set them to avoid "hogging" the node, but this causes throttling even when the node has spare CPU sitting idle, silently degrading performance without any visible error. Many teams now use only CPU requests and skip CPU limits, relying on requests + Quality of Service classes for fairness.
- **Setting memory limits too low** will get your pods OOMKilled outright, so it's worth headroom based on real observed usage (via `kubectl top` or metrics-server).
- **QoS classes** are derived from requests/limits:
    - `Guaranteed`: requests == limits for both CPU and memory (highest priority, last to be evicted)
    - `Burstable`: requests < limits (or only requests set)
    - `BestEffort`: no requests/limits set at all (first to be evicted under pressure)

## Two ways to set the namespace

**Option 1: In the YAML itself**

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: cpu-mem-limits
  namespace: team-a    # ← set here
spec:
  ...
```

```bash
kubectl apply -f limit-range.yaml
```

**Option 2: Via the `--namespace` flag**

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: cpu-mem-limits    # ← no namespace field here
spec:
  ...
```

```bash
kubectl apply -f limit-range.yaml --namespace=team-a
```

Both work. If the YAML has a `namespace:` field, that value takes precedence — the `--namespace` flag mostly matters when the file _doesn't_ specify one.

## Two small corrections to your command

1. **Use `apply`, not `create`**, as your default habit. `create` fails with an "already exists" error if you run it twice (e.g. re-running after a small YAML tweak). `apply` is idempotent — it creates on first run, updates on subsequent runs. `create` isn't wrong, it's just less forgiving.
    
2. `-f` still works fine with either — the flag order doesn't matter:
    

```bash
kubectl apply -f limit-range.yaml -n team-a
kubectl apply -f resourcequota.yaml -n team-a
```

(`-n` is just shorthand for `--namespace`.)

## Verifying it landed in the right namespace

```bash
kubectl get limitrange -n team-a
kubectl get resourcequota -n team-a
kubectl describe limitrange cpu-mem-limits -n team-a
kubectl describe resourcequota team-quota -n team-a
```

If you apply without specifying a namespace at all (no `namespace:` in YAML and no `-n` flag), it goes into whatever your current `kubectl` context's default namespace is (usually `default`) — which is a common source of "why isn't my quota working" confusion, since it silently lands in the wrong place instead of erroring out.

**Best practice:** put `namespace:` explicitly in the YAML rather than relying on the flag. It makes the manifest self-documenting and prevents accidental misapplication if someone runs it without `-n`.


To see the reason for failing pod (Last State Reason)
```yaml
controlplane ~ ➜  k describe pod elephant | grep -A5 "Last State"
    Last State:     Terminated
      Reason:       OOMKilled
      Exit Code:    137
      Started:      Sat, 15 Aug 2026 14:25:54 +0000
      Finished:     Sat, 15 Aug 2026 14:25:54 +0000
    Ready:          False
```
