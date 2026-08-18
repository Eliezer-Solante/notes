![[Pasted image 20260818111336.png]]

![[Pasted image 20260818114141.png]]
![[Pasted image 20260818114210.png]]

![[Pasted image 20260818114447.png]]

![[Pasted image 20260818114516.png]]

![[Pasted image 20260818114531.png]]
## Blue/Green Deployment in Kubernetes

Blue/Green deployment is a release strategy where you run **two identical production environments** — "Blue" (current live version) and "Green" (new version) — and switch traffic between them instead of updating in place.

### How it works in Kubernetes

1. **Deploy "Green"** alongside the existing "Blue" deployment, using a separate set of pods (usually a different `Deployment` with a distinguishing label, e.g. `version: green`).
2. **Test Green internally** — route test traffic to it via a separate Service or port-forward, without exposing it to users yet.
3. **Switch traffic** by updating the main `Service`'s selector to match the Green deployment's labels instead of Blue's. This is typically instant since Kubernetes just re-routes via the Service's endpoint list.
4. **Monitor** the new version under real traffic.
5. **Rollback** (if needed) is just as fast — flip the selector back to Blue.
6. **Decommission Blue** once you're confident Green is stable (or keep it around as a fallback for the next cycle).

### Typical implementation

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-app
spec:
  selector:
    app: my-app
    version: blue   # change to "green" to cut over
  ports:
    - port: 80
```

Two Deployments (`my-app-blue`, `my-app-green`) run concurrently, each with distinct labels, and the Service selector is the single point that controls which one receives live traffic.

### Pros

- **Instant cutover and rollback** — just change a selector or ingress rule.
- **Zero downtime** — new version is fully warmed up before receiving traffic.
- **Easy full-environment testing** before exposure.

### Cons

- **Resource cost** — you need double the capacity during the transition.
- **Stateful workloads** are tricky — database schema changes/migrations must stay compatible with both versions during the overlap.
- Kubernetes has no built-in "Blue/Green" primitive — you're composing it from Deployments + Services (or using tools like **Argo Rollouts**, **Flagger**, or a **service mesh/ingress controller** for more automated traffic shifting, including canary-style gradual cutover).

### Contrast with Rolling Update (Kubernetes' default)

Rolling updates replace pods incrementally within one Deployment — cheaper on resources but with a mixed-version window and slower rollback. Blue/Green trades resource efficiency for cleaner, instant, all-or-nothing cutovers.