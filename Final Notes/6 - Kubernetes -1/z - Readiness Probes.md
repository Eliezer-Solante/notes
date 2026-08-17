![[Pasted image 20260817145444.png]]

sample
![[Pasted image 20260817150432.png]]

![[Pasted image 20260817150522.png]]

### Readiness Probe

**Question it answers:** "Is this container ready to accept traffic right now?"

**What happens on failure:** The pod is removed from the Service's list of endpoints — so no traffic is routed to it — but the container is **not** restarted. It's added back once the probe succeeds again.

**Purpose:** Handles temporary states where the app is alive and running but not yet able to serve requests properly — during startup while loading config or warming a cache, or temporarily during runtime if it's overloaded or waiting on a dependency (like a database) to become available.

**Example use case:** An app that takes 30 seconds to load a large dataset into memory before it can serve requests — you don't want traffic hitting it during that window.

```yaml
readinessProbe:
  httpGet:
    path: /ready
    port: 8080
  initialDelaySeconds: 5
  periodSeconds: 10
  failureThreshold: 3
```