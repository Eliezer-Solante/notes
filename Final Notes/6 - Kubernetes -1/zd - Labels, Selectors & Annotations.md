
![[Pasted image 20260817162934.png]]



![[Pasted image 20260817162908.png]]

![[Pasted image 20260817162920.png]]

![[Pasted image 20260817163321.png]]
![[Pasted image 20260817163329.png]]


![[Pasted image 20260817163519.png]]

![[Pasted image 20260817163613.png]]


# Labels, Selectors, and Annotations

## Labels

Key-value pairs attached to Kubernetes objects (Pods, Services, Deployments, etc.) used to **identify and organize** resources.

```yaml
metadata:
  labels:
    app: my-app
    env: production
    tier: backend
```

- Meant to be **meaningful and queryable** — they identify attributes that matter for grouping/selecting objects.
- Multiple objects can share the same label, and one object can have multiple labels.
- Commonly used to distinguish environments, app versions, tiers, or ownership.

## Selectors

The mechanism used to **query/filter objects by their labels**. Selectors are how one resource finds another.

```yaml
# A Service selecting Pods labeled app=my-app
selector:
  app: my-app
```

- **Equality-based**: `env=production`, `tier!=frontend`
- **Set-based**: `env in (production, staging)`, `tier notin (frontend)`

Used heavily by:

- **Services** — to know which Pods to route traffic to
- **Deployments/ReplicaSets** — to know which Pods they own/manage
- **NetworkPolicies** — to define which Pods traffic rules apply to
- `kubectl get pods -l app=my-app` — for querying via CLI

**Key point:** labels + selectors are the glue that connects otherwise independent objects (e.g., a Service has no direct pointer to Pods — it just matches on labels).

## Annotations

Also key-value pairs, but for **non-identifying metadata** — not used for selection.

```yaml
metadata:
  annotations:
    description: "Handles payment processing"
    kubernetes.io/change-cause: "Updated to v2.3.1"
    prometheus.io/scrape: "true"
```

- Not queryable/selectable — Kubernetes won't filter objects by annotations.
- Used for arbitrary, often larger or more free-form data: build info, contact details, tool-specific config (e.g., ingress controllers, Prometheus scraping hints), changelogs, links to docs.

## Quick Comparison

![[Pasted image 20260817163641.png]]

**Rule of thumb:** if you need to _find or group_ objects by it → label. If it's just _informational_ or used by a specific tool/controller reading its own annotation → annotation.
