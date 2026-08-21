![[Pasted image 20260821204644.png]]
![[Pasted image 20260821204953.png]]
![[Pasted image 20260821205151.png]]
![[Pasted image 20260821205432.png]]

![[Pasted image 20260821205420.png]]
![[Pasted image 20260821205532.png]]
![[Pasted image 20260821205656.png]]
![[Pasted image 20260821210951.png]]


---

# Validating vs Mutating Admission Controllers

Admission controllers split into two categories based on **what they're allowed to do** to a request as it passes through the pipeline:

```
User → kubelet → Authentication → Authorization → Admission Controllers → Create Pod/PVC
```

## Validating Admission Controllers

These can only **accept or reject** the request — they cannot change the object.

```
kubelet → Authentication → Authorization → [Admission Controllers: NamespaceExists] → Create Pod
```

Example: `NamespaceExists` simply checks if the target namespace exists. If not — reject. It never modifies the incoming object, it only says yes/no.

## Mutating Admission Controllers

These can **modify the object** before it's persisted — filling in defaults, injecting fields, etc.

```
kubelet → Authentication → Authorization → [Admission Controllers: DefaultStorageClass] → Create PVC
```

Example: a PVC is submitted **without** `storageClassName` set:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: myclaim
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 500Mi
  storageClassName: default    # ← injected by DefaultStorageClass
```

Even though the manifest didn't specify a storage class, `DefaultStorageClass` **mutates** the object and injects one automatically:

```bash
$ kubectl describe pvc myclaim
Name:          myclaim
Namespace:     default
StorageClass:  default        # ← auto-filled by the mutating controller
Status:        Pending
```

## Built-in Controllers, Categorized

```
Admission Controllers
├── Mutating
│   ├── NamespaceAutoProvision
│   └── (many built-ins mutate: DefaultStorageClass, AlwaysPullImages...)
├── Validating
│   ├── NamespaceExists
│   └── (many built-ins validate: EventRateLimit...)
└── Generic hooks (can do either)
    ├── MutatingAdmissionWebhook
    └── ValidatingAdmissionWebhook
```

Important: **mutating controllers always run before validating ones** — so any fields a mutating controller injects (like a default storage class) are then checked by validating controllers, not the other way around.

## Webhooks — Extending Admission with Custom Logic

The built-in plugins cover common cases, but for **custom business rules**, you use `MutatingAdmissionWebhook` / `ValidatingAdmissionWebhook` — these forward the request to an **external webhook server** you write and deploy yourself.

### The Request/Response Contract

The API server sends an `AdmissionReview` object to your webhook, and expects one back:

**Request sent to webhook:**

```json
{
  "apiVersion": "admission.k8s.io/v1",
  "kind": "AdmissionReview",
  "request": {
    "uid": "705ab4f5-6393-11e8-b7cc-42010a800002",
    "kind": {"group": "autoscaling", "version": "v1", "kind": "Scale"},
    "resource": {"group": "apps", "version": "v1", "resource": "deployments"},
    "subResource": "scale"
  }
}
```

**Response expected back:**

```json
{
  "apiVersion": "admission.k8s.io/v1",
  "kind": "AdmissionReview",
  "response": {
    "uid": "<value from request.uid>",
    "allowed": true
  }
}
```

## Building a Webhook Server: Two Steps

### Step 1 — Deploy the Webhook Server

Your server implements two endpoints — one for validation, one for mutation.

**`/validate` (example in Python):**

```python
@app.route("/validate", methods=["POST"])
def validate():
    object_name = request.json["request"]["object"]["metadata"]["name"]
    user_name = request.json["request"]["userInfo"]["name"]
    status = True
    if object_name == user_name:
        message = "You can't create objects with your own name"
        status = False
    return jsonify({
        "response": {
            "allowed": status,
            "uid": request.json["request"]["uid"],
            "status": {"message": message}
        }
    })
```

This rejects a request only if the object's name matches the requesting user's name — pure validation, no mutation.

**`/mutate`:**

```python
@app.route("/mutate", methods=["POST"])
def mutate():
    user_name = request.json["request"]["userInfo"]["name"]
    patch = [{"op": "add", "path": "/metadata/labels/users", "value": user_name}]
    return jsonify({
        "response": {
            "allowed": True,
            "uid": request.json["request"]["uid"],
            "patch": base64.b64encode(patch),
            "patchtype": "JSONPatch"
        }
    })
```

This automatically **injects a label** (`users: <requesting-user>`) onto every object created — a JSON Patch, base64-encoded, tells the API server exactly what to change.

> Note the key structural difference: `/validate` returns just `allowed` + optional `status.message`; `/mutate` additionally returns a `patch` + `patchtype` describing the mutation to apply.

### Step 2 — Register the Webhook with the Cluster

Deploy your webhook server as a Service inside the cluster, then register it via a `ValidatingWebhookConfiguration` (or `MutatingWebhookConfiguration`):

```
Kubernetes Cluster
└── webhook-deployment
    └── webhook-service → Admission Webhook Server
```

```yaml
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingWebhookConfiguration
metadata:
  name: "pod-policy.example.com"
webhooks:
- name: "pod-policy.example.com"
  clientConfig:
    service:
      namespace: "webhook-namespace"
      name: "webhook-service"
    caBundle: "Ci0tLS0tQk......tLS0K"     # CA cert to verify the webhook's TLS
  rules:
  - apiGroups:   [""]
    apiVersions: ["v1"]
    operations:  ["CREATE"]
    resources:   ["pods"]
    scope:       "Namespaced"
```

This tells the API server: _"whenever a Pod is CREATEd in a namespace, call `webhook-service` and wait for its verdict before proceeding."_


TASK SAMPLE

Creating a TLS secret
```bash
kubectl -n webhook-demo create secret tls webhook-server-tls \
    --cert "/root/keys/webhook-server-tls.crt" \
    --key "/root/keys/webhook-server-tls.key"
```


