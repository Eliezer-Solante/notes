to get the actual token
![[Pasted image 20260815235631.png]]

creating using imperative command
![[Pasted image 20260815235801.png]]


creating using declarative (YAML)
![[Pasted image 20260815235819.png]]
Inject to the Pod
![[Pasted image 20260815235830.png]]


To disable automatic creation of token in the pod
service-level
![[Pasted image 20260816000000.png]]

pod-level
![[Pasted image 20260816000027.png]]


Creating token (Manual)
![[Pasted image 20260816000522.png]]

![[Pasted image 20260816000634.png]]

![[Pasted image 20260816000708.png]]

![[Pasted image 20260816000721.png]]

![[Pasted image 20260816000826.png]]


Sample Situation
## Notes: Disabling ServiceAccount Token Auto-Mount

**Goal:** stop a ServiceAccount's token from being auto-mounted into pods, then explicitly mount it via a projected volume for finer control (shorter-lived, scoped token).

---

### Part 1 — Disable `automountServiceAccountToken` on the ServiceAccount

**Option A: `kubectl edit`**

```bash
kubectl edit sa dashboard-sa
```

Add at the top level:

```yaml
apiVersion: v1
automountServiceAccountToken: false
kind: ServiceAccount
metadata:
  name: dashboard-sa
  namespace: default
```

**Option B: `kubectl patch`** (faster, no editor)

```bash
kubectl patch sa dashboard-sa -p '{"automountServiceAccountToken": false}'
```

---

### Part 2 — Configure projected volume on the Deployment

Edit the deployment manifest:

```bash
vim ~/web-dashboard/deployment.yaml
```

Add to `spec.template.spec`:

- `automountServiceAccountToken: false` — disable default mount at the pod level too
- A **projected volume** named `token`, sourced from `serviceAccountToken`
- A **volumeMount** in the container at `/var/run/secrets/kubernetes.io/serviceaccount/`, `readOnly: true`

**Final manifest:**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-dashboard
  namespace: default
  labels:
    name: web-dashboard
spec:
  replicas: 1
  selector:
    matchLabels:
      name: web-dashboard
  template:
    metadata:
      labels:
        name: web-dashboard
    spec:
      containers:
      - name: web-dashboard
        image: gcr.io/kodekloud/customimage/my-kubernetes-dashboard
        ports:
        - containerPort: 8080
        env:
        - name: PYTHONUNBUFFERED
          value: "1"
        volumeMounts:
        - mountPath: /var/run/secrets/kubernetes.io/serviceaccount/
          name: token
          readOnly: true
      serviceAccountName: dashboard-sa
      automountServiceAccountToken: false
      volumes:
      - name: token
        projected:
          sources:
          - serviceAccountToken:
              path: token
```

Apply and verify:

```bash
kubectl apply -f ~/web-dashboard/deployment.yaml
kubectl get pods
```

Verify if token is mounted properly
```bash
cat kubectl exec $(kubectl get pod -l name=web-dashboard -o jsonpath='{.items[0].metadata.name}') -- ls /var/run/secrets/kubernetes.io/serviceaccount/
```

---

### Key takeaway

Two separate `automountServiceAccountToken: false` settings — one on the **SA**, one on the **pod spec** — both need to be off if you want to fully replace the default auto-mount with your own controlled projected-volume token. The projected volume then gives you a token you control (path, audience, expiration), instead of the default long-lived-style mount.