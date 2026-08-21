![[Pasted image 20260821200923.png]]
![[Pasted image 20260821201008.png]]
![[Pasted image 20260821201035.png]]
![[Pasted image 20260821201203.png]]
![[Pasted image 20260821201251.png]]
![[Pasted image 20260821201306.png]]
![[Pasted image 20260821201507.png]]


![[Pasted image 20260821201625.png]]

![[Pasted image 20260821201715.png]]
![[Pasted image 20260821201748.png]]


---

# Admission Controllers

## The Full Request Pipeline

Every request to Kubernetes goes through **three gates** before anything is actually created:

```
User → kubelet → Authentication → Authorization → Admission Controllers → Create Pod
```

- **Authentication** — who are you? (verified via `~/.kube/config` certs/tokens)
- **Authorization** — are you allowed to do this? (RBAC roles/rolebindings)
- **Admission Controllers** — the request passed auth, but should it _actually_ be allowed/modified before it's persisted?

## Why Admission Controllers Exist

RBAC can only answer yes/no questions about **actions on resource types** — it can't enforce business logic or intercept/modify the object itself.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: developer
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["list", "get", "create", "update", "delete"]
```

RBAC **can** enforce:

- ✅ Can list/create/delete PODs/Deployments/Services
- ✅ Can create pods named `blue` or `orange` (via `resourceNames`)
- ✅ Can create pods within a namespace

RBAC **cannot** enforce:

- ❌ Only permit images from a certain registry
- ❌ Do not permit `runAsRoot`
- ❌ Only permit certain capabilities

```yaml
# web-pod.yaml — RBAC alone can't stop this:
apiVersion: v1
kind: Pod
metadata:
  name: web-pod
spec:
  containers:
  - name: ubuntu
    image: ubuntu:latest        # ❌ untrusted registry
    command: ["sleep", "3600"]
    securityContext:
      runAsUser: 0               # ❌ root
      capabilities:
        add: ["MAC_ADMIN"]       # ❌ dangerous capability
```

This is the gap **Admission Controllers** fill — plugins that intercept requests **after** auth/authz but **before** the object is persisted to etcd, and can **validate**, **mutate**, or **reject** them.

## What Admission Controllers Do

They sit in the pipeline as a set of plugins:

```
Admission Controllers
├── AlwaysPullImages
├── DefaultStorageClass
├── EventRateLimit
├── NamespaceExists / NamespaceAutoProvision
└── Many more...
```

### Example: `NamespaceExists`

If enabled, this controller **rejects** pod creation in a namespace that doesn't exist yet:

```bash
$ kubectl run nginx --image nginx --namespace blue
Error from server (NotFound): namespaces "blue" not found
```

### Example: `NamespaceAutoProvision`

If this controller is enabled _instead_, the same command **auto-creates** the namespace rather than rejecting the request:

```bash
$ kubectl run nginx --image nginx --namespace blue
Pod/nginx created!

$ kubectl get namespaces
NAME          STATUS   AGE
blue          Active   3m
default       Active   23m
kube-public   Active   24m
kube-system   Active   24m
```

This shows the two categories of admission controllers:

- **Validating** — accept or reject the request as-is (e.g. `NamespaceExists`)
- **Mutating** — modify the request before it's persisted (e.g. `NamespaceAutoProvision`, `DefaultStorageClass`, `AlwaysPullImages`)

Some controllers do both (mutate first, then validate) — this is how `MutatingAdmissionWebhook` and `ValidatingAdmissionWebhook` work with external policy engines like OPA.

## Viewing Enabled Admission Controllers

```bash
$ kube-apiserver -h | grep enable-admission-plugins
  --enable-admission-plugins strings
    admission plugins that should be enabled in addition to default enabled ones
    (NamespaceLifecycle, LimitRanger, ServiceAccount, TaintNodesByCondition, Priority,
    DefaultTolerationSeconds, DefaultStorageClass, StorageObjectInUseProtection,
    PersistentVolumeClaimResize, RuntimeClass, CertificateApproval, CertificateSigning,
    CertificateSubjectRestriction, DefaultIngressClass, MutatingAdmissionWebhook,
    ValidatingAdmissionWebhook, ResourceQuota).
    Comma-delimited list: AlwaysAdmit, AlwaysDeny, AlwaysPullImages, CertificateApproval,
    CertificateSigning, CertificateSubjectRestriction, DefaultIngressClass,
    DefaultStorageClass, DefaultTolerationSeconds, DenyEscalatingExec, DenyExecOnPrivileged,
    EventRateLimit, ExtendedResourceToleration, ImagePolicyWebhook,
    LimitPodHardAntiAffinityTopology, LimitRanger, MutatingAdmissionWebhook,
    NamespaceAutoProvision, NamespaceExists, NamespaceLifecycle, NodeRestriction, ...
    TaintNodesByCondition, ValidatingAdmissionWebhook.
    The order of plugins in this flag does not matter.
```

If running kube-apiserver as a static pod (kubeadm setup):

```bash
kubectl exec kube-apiserver-controlplane -n kube-system -- kube-apiserver -h | grep enable-admission-plugins
```

## Enabling / Disabling Admission Controllers

**Option A — systemd service** (`kube-apiserver.service`):
located at `/etc/kubernetes/manifests/kube-apiserver.yaml`
```bash
ExecStart=/usr/local/bin/kube-apiserver \
  --advertise-address=${INTERNAL_IP} \
  --allow-privileged=true \
  --apiserver-count=3 \
  --authorization-mode=Node,RBAC \
  --bind-address=0.0.0.0 \
  --enable-swagger-ui=true \
  --etcd-servers=https://127.0.0.1:2379 \
  --event-ttl=1h \
  --runtime-config=api/all \
  --service-cluster-ip-range=10.32.0.0/24 \
  --service-node-port-range=30000-32767 \
  --v=2 \
  --enable-admission-plugins=NodeRestriction,NamespaceAutoProvision \
  --disable-admission-plugins=DefaultStorageClass
```

**Option B — kubeadm static pod manifest** (`/etc/kubernetes/manifests/kube-apiserver.yaml`):

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: kube-apiserver
  namespace: kube-system
spec:
  containers:
  - command:
    - kube-apiserver
    - --authorization-mode=Node,RBAC
    - --advertise-address=172.17.0.107
    - --allow-privileged=true
    - --enable-bootstrap-token-auth=true
    - --enable-admission-plugins=NodeRestriction,NamespaceAutoProvision
    image: k8s.gcr.io/kube-apiserver-amd64:v1.11.3
    name: kube-apiserver
```

Editing this file triggers the kubelet to automatically restart the static pod with the new flags — no separate restart command needed.

## Quick Reference

|Flag|Purpose|
|---|---|
|`--enable-admission-plugins`|Comma-separated list to enable in addition to defaults|
|`--disable-admission-plugins`|Comma-separated list to turn off|

```bash
# Check which plugins are active right now
ps -ef | grep kube-apiserver | grep admission-plugins
```

Together, **Authentication → Authorization → Admission Controllers** form the complete defense-in-depth chain: prove identity → check permission → enforce policy/mutate object — before anything ever reaches etcd.



Good catch — that's an important correction, since I presented `NamespaceExists`/`NamespaceAutoProvision` as if they were still standalone options you'd toggle today.

## The Correction

`NamespaceExists` and `NamespaceAutoProvision` are **deprecated/removed** in modern Kubernetes. Their functionality was folded into a single controller:

```
NamespaceLifecycle
```

This is why it appears in the **default-enabled** list in the help output shown earlier:

```
--enable-admission-plugins strings
    admission plugins that should be enabled in addition to default enabled ones
    (NamespaceLifecycle, LimitRanger, ServiceAccount, TaintNodesByCondition, Priority,
    DefaultTolerationSeconds, DefaultStorageClass, ...)
```

`NamespaceLifecycle` is on by default and cannot practically be disabled in a working cluster.

## What `NamespaceLifecycle` Does

It combines the old behaviors into one controller that:

1. **Rejects** requests to create resources in a namespace that doesn't exist or is being terminated — same effect as the old `NamespaceExists`:

```bash
$ kubectl run nginx --image nginx --namespace blue
Error from server (NotFound): namespaces "blue" not found
```

2. **Prevents deletion** of the three default system namespaces (`default`, `kube-system`, `kube-public`), which the old controllers didn't handle.
    
3. Ensures a namespace in `Terminating` state can't have new objects created in it.
    

**Note:** it does **not** auto-create namespaces the way `NamespaceAutoProvision` used to. If you want a namespace to auto-create on first use, that behavior is gone — you must create the namespace explicitly:

```bash
kubectl create namespace blue
kubectl run nginx --image nginx --namespace blue
```

## Revised Default-Enabled List (typical modern cluster)

```
NamespaceLifecycle, LimitRanger, ServiceAccount, TaintNodesByCondition, Priority,
DefaultTolerationSeconds, DefaultStorageClass, StorageObjectInUseProtection,
PersistentVolumeClaimResize, RuntimeClass, CertificateApproval, CertificateSigning,
CertificateSubjectRestriction, DefaultIngressClass, MutatingAdmissionWebhook,
ValidatingAdmissionWebhook, ResourceQuota
```

## Updated Quick Reference

|Old (deprecated)|Replaced by|
|---|---|
|`NamespaceExists`|`NamespaceLifecycle`|
|`NamespaceAutoProvision`|`NamespaceLifecycle` (auto-provision behavior removed entirely)|

```bash
# Verify what's actually enabled on your cluster
kube-apiserver -h | grep enable-admission-plugins
kubectl exec kube-apiserver-controlplane -n kube-system -- kube-apiserver -h | grep enable-admission-plugins
```

Thanks for flagging that — good to keep the terminology current since exam material (e.g. CKA) sometimes still references the older names even though they no longer exist as separate flags.



TASK SAMPLE

# Task: Enable `ImagePolicyWebhook` Admission Plugin

**Goal:** Reconfigure the API server to enable the `ImagePolicyWebhook` admission plugin and make sure it can access its config files.

## Checklist

- [x] `ImagePolicyWebhook` admission plugin enabled on kube-apiserver?
- [x] `--admission-control-config-file` flag set on kube-apiserver?
- [x] `imgvalidation` volume mounted in kube-apiserver?

## Steps

**1. Back up the manifest first, then edit it:**

```bash
cp /etc/kubernetes/manifests/kube-apiserver.yaml /opt/kube-apiserver.yaml.bak
vi /etc/kubernetes/manifests/kube-apiserver.yaml
```

**2. Enable the admission plugin** — add `ImagePolicyWebhook` alongside existing plugins:

```yaml
    - --enable-admission-plugins=NodeRestriction,ImagePolicyWebhook
```

**3. Point to the admission control config file:**

```yaml
    - --admission-control-config-file=/etc/kubernetes/imgvalidation/admission-configuration.yaml
```

**4. Mount the `imgvalidation` directory so the config file is actually reachable inside the container:**

Under `volumes`:

```yaml
    - name: imgvalidation
      hostPath:
        path: /etc/kubernetes/imgvalidation
        type: Directory
```

Under `volumeMounts`:

```yaml
    - name: imgvalidation
      mountPath: /etc/kubernetes/imgvalidation
      readOnly: true
```

**5. Save and verify the API server restarted cleanly** (kubelet auto-restarts static pods on manifest change):

```bash
kubectl get pods -n kube-system
```

## Notes to self

- No need to manually restart anything — editing a static pod manifest in `/etc/kubernetes/manifests/` triggers the kubelet to recreate the pod automatically.
- If the apiserver pod doesn't come back (crash loop / missing), check `crictl logs` on the node or `kubectl describe pod kube-apiserver-<node> -n kube-system` — likely causes: bad YAML indentation, config file path typo, or the `admission-configuration.yaml` referencing a webhook that isn't reachable.
- Keep the `.bak` file until confirmed working — fastest rollback is just copying it back over the manifest.