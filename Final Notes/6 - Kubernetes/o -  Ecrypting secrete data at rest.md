## Encrypting Secret Data at Rest — Full Discussion with Verification

### The problem this solves

We've established that Secrets are **base64-encoded, not encrypted**, in `data`. But there's a layer beneath even that: every Secret object gets persisted into **etcd**, Kubernetes' backing key-value store, and by default etcd stores that same base64 data as-is — no encryption applied. Anyone who gains access to etcd's raw storage (stolen disk, leaked backup, unauthorized direct access) can read every Secret in the cluster, completely bypassing the Kubernetes API and RBAC entirely.

```
kubectl create secret ... → API server → etcd (plaintext by default, unless configured otherwise)
```

This is a distinct attack surface from RBAC. RBAC governs _who can ask the API_ for a Secret. Encryption-at-rest governs _what's actually readable if someone bypasses the API and reads storage directly_.

### How to check the current status — before doing anything else

Since this is a control-plane setting rather than a normal Kubernetes object, there's no simple `kubectl get` for it. The checks differ depending on your access level.

**If you have control plane node access:**

```bash
# check if the API server was started with encryption config
ps -ef | grep kube-apiserver | grep encryption-provider-config

# for kubeadm-style clusters (static pod manifest)
cat /etc/kubernetes/manifests/kube-apiserver.yaml | grep encryption-provider-config
```

No flag present → encryption is **not enabled** (default `identity`/plaintext provider is in use).

**If the flag exists, check what it actually points to:**

```bash
cat /etc/kubernetes/enc/encryption-config.yaml
```

```yaml
resources:
  - resources:
      - secrets
    providers:
      - aescbc:
          keys: [...]
      - identity: {}
```

The **first provider listed is what applies to new writes** — if `identity` comes first, encryption is effectively off regardless of what else is in the file.

**The most reliable check — inspect the raw bytes directly in etcd:**

```bash
ETCDCTL_API=3 etcdctl \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  get /registry/secrets/default/my-secret --print-value-only
```

|Output|Meaning|
|---|---|
|`k8s:enc:aescbc:v1:key1:...` (opaque binary)|✅ Encrypted|
|Readable `apiVersion`/`kind: Secret`/base64 `data` structure|❌ Plaintext|

This is the definitive test — it checks actual stored bytes, not just whether a config file exists.

**On managed cloud clusters (EKS/GKE/AKS)** — you won't have etcd access, so check the provider's own tooling instead:

```bash
aws eks describe-cluster --name my-cluster --query "cluster.encryptionConfig"
gcloud container clusters describe my-cluster --format="value(databaseEncryption)"
```

**With only standard `kubectl` access** — you genuinely cannot check this yourself. It's deliberately outside the scope of what the API exposes to regular users; you'd need to ask the cluster admin.

### How to enable it

Once you've confirmed it's off, create an `EncryptionConfiguration`:

```yaml
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
  - resources:
      - secrets
    providers:
      - aescbc:
          keys:
            - name: key1
              secret: <base64-encoded 32-byte key>
      - identity: {}
```

Reference it in the API server's startup flags:

```
--encryption-provider-config=/etc/kubernetes/enc/encryption-config.yaml
```

|Provider|Notes|
|---|---|
|`identity`|No encryption (default)|
|`aescbc`|AES-CBC, local key file|
|`aesgcm`|AES-GCM, similar|
|`secretbox`|XSalsa20-Poly1305|
|`kms`|External key management (AWS/GCP/Azure KMS, Vault) — recommended for production, key never touches cluster disk|

### The gotcha: enabling it doesn't retroactively encrypt existing Secrets

Only Secrets **written after** the config change get encrypted. Force re-encryption of everything already stored:

```bash
kubectl get secrets --all-namespaces -o json | kubectl replace -f -
```

### Verify it worked — re-run the same etcd check

```bash
ETCDCTL_API=3 etcdctl get /registry/secrets/default/my-secret --print-value-only
```

If it now shows the `k8s:enc:aescbc:v1:...` prefix instead of readable structure, encryption is confirmed active for that Secret.

### The three-layer summary

|Layer|Protects against|How to verify|
|---|---|---|
|Base64 (`data` field)|Nothing — it's an encoding|`kubectl get secret -o yaml` shows it's reversible instantly|
|RBAC|Users/service accounts calling the API|`kubectl auth can-i get secrets`|
|Encryption at rest|Direct access to etcd storage/backups|`etcdctl get` on raw stored bytes, or API server flag/config inspection|

The core workflow, in order: **check current status first** (via etcd inspection or cloud provider tooling) → **enable** `EncryptionConfiguration` if it's off → **re-encrypt existing Secrets** with the bulk replace command → **verify** with the same etcd check you started with, confirming the output changed from readable to opaque.