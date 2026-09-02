# Kubernetes Troubleshooting Scenarios — Summary & Solutions

## Scenario 01: Blue-Green Cutover Stuck on Old Color

**Namespace:** `retail-ops`

**Context:** The retail team ran a blue-green cutover: `checkout-blue` (v1) is the old Deployment, `checkout-green` (v2) is the new one, and both are meant to sit behind Service `checkout-service`.

**Concept:** A Service doesn't know about Deployments — it just matches its `spec.selector` against Pod labels and routes to whatever matches. If the selector doesn't match, the pod is invisible to it, no matter how healthy it is.

**Task:** Compare the Service's `spec.selector` against the labels on both Deployments' pods, and update `checkout-service`'s selector so `version` reads `green` instead of `blue`, completing the cutover.

**Issue found:** `checkout-service`'s `spec.selector` still targeted `version: blue`, so it never matched the healthy `checkout-green` pods.

**Solution:**

```bash
kubectl patch service checkout-service -n retail-ops \
  --type='json' \
  -p='[{"op": "replace", "path": "/spec/selector/version", "value": "green"}]'
```

Verify the Service's Endpoints now match the green pods' IPs:

```bash
kubectl get endpoints checkout-service -n retail-ops
```

---

## Scenario 02: Job Not Honoring Parallelism

**Namespace:** `media-ops`

**Context:** Two Jobs are running. `thumbnail-batch` is already working as expected, processing 6 items with 3 running concurrently. Job `image-resize-batch` is meant to process 5 images with up to 2 running concurrently, but only ever shows 1.

**Concept:** A Job's `completions` (total successful runs needed) and `parallelism` (how many pods run at once) are two separate settings — leaving one out doesn't error, it just silently defaults.

**Task:** Compare Job `image-resize-batch` to the already-working Job `thumbnail-batch`, and update Job `image-resize-batch`'s configuration to make it work as expected.

**Issue found:** `image-resize-batch` omitted `spec.parallelism`, which silently defaults to `1` instead of erroring — so only one pod ran at a time instead of the intended 2.

**Solution:**

```bash
kubectl patch job image-resize-batch -n media-ops \
  --type='json' \
  -p='[{"op": "add", "path": "/spec/parallelism", "value": 2}]'
```

Verify:

```bash
kubectl get pods -n media-ops -l job-name=image-resize-batch
```

---

## Scenario 03: Invoice API Has No Traffic Restrictions

**Namespace:** `payments-ops`

**Context:** The `invoice-api` pod currently has no NetworkPolicy applied to it at all — it's reachable from any pod, in any namespace, on the cluster. Security wants it locked down so that only `billing-ui` pods in namespace `billing-ops` can reach it, on port `8080`.

**Concept:** A NetworkPolicy is a firewall rule for pods, and it's opt-in per pod: with zero policies selecting it, a pod allows all traffic from anywhere. The moment any policy's `podSelector` matches a pod, that pod flips to "deny everything except what's explicitly allowed" — so writing one correctly-scoped policy both grants the access you want and locks out everything else, in a single step.

**Task:** Create a new NetworkPolicy in `payments-ops` that selects `app: invoice-api` pods, with one ingress rule allowing traffic on `port: 8080` from pods labeled `app: billing-ui` that live in namespace `billing-ops`. (Requires one `from` entry with both a `namespaceSelector` matching `billing-ops` and a `podSelector` matching `app: billing-ui`, together in the same entry, so the rule means "this pod, in this namespace" rather than "either of these.")

**Issue found:** No NetworkPolicy selected `invoice-api`, so it was fully exposed.

**Solution:**

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: invoice-api-allow-billing-ui
  namespace: payments-ops
spec:
  podSelector:
    matchLabels:
      app: invoice-api
  policyTypes:
    - Ingress
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: billing-ops
          podSelector:
            matchLabels:
              app: billing-ui
      ports:
        - protocol: TCP
          port: 8080
```

**Key detail:** `namespaceSelector` and `podSelector` must sit in the **same** `from` list entry (AND condition) — not as two separate entries (which would become OR, and far too permissive).

---

## Scenario 04: Ingress Points at a Service That Doesn't Exist

**Namespace:** `storefront-ops`

**Context:** Ingress `storefront-ingress` is meant to route `storefront.example.com` traffic to Service `storefront-svc`.

**Concept:** Kubernetes doesn't check that an Ingress's backend service name actually exists when you `apply` the manifest — the Ingress object is accepted either way. A typo'd service name only shows up once real traffic tries to flow through and the ingress controller can't find a matching Service to send it to.

**Task:**

1. Check the Service in namespace `storefront-ops`.
2. Compare its name against the Service name referenced in the Ingress's backend (`spec.rules[].http.paths[].backend.service.name`).
3. Check for any mismatch between the two, and fix the Ingress so it references the correct Service name.

**Issue found:** The Ingress's backend name didn't match the actual Service name.

**Solution:**

```bash
kubectl patch ingress storefront-ingress -n storefront-ops \
  --type='json' \
  -p='[{"op": "replace", "path": "/spec/rules/0/http/paths/0/backend/service/name", "value": "storefront-svc"}]'
```

Verify:

```bash
kubectl get endpoints storefront-svc -n storefront-ops
```

---

## Scenario 05: Cache Rebuilt on Every Restart

**Namespace:** `content-ops`

**Context:** `quote-cache-warmer` writes a cache file to `/var/cache/quotes/index.db` the first time it starts. Rebuilding it takes about 90 seconds, so the team wants that cache to survive a restart instead of being rebuilt every time.

**Concept:** Anything a container writes without a volume disappears the moment the container restarts, even though the Pod itself keeps running. Only a volume makes written files survive a restart.

**Task:** Check the Deployment's Pod template's `volumes` and the container's `volumeMounts` to see whether a mount path of `/var/cache/quotes` is configured at all. Add an `emptyDir` volume, and add a matching `volumeMounts` entry to the existing container (alongside its current `image`/`command`, not replacing them) so it mounts at that path.

**Issue found:** No volume was configured at all — the cache file was written to the container's ephemeral writable layer.

**Solution:**

```yaml
containers:
  - name: quote-cache-warmer
    volumeMounts:
      - name: quote-cache
        mountPath: /var/cache/quotes
volumes:
  - name: quote-cache
    emptyDir: {}
```

**Note:** `emptyDir` survives container restarts within the same Pod, but not Pod deletion/rescheduling — a `PersistentVolumeClaim` would be needed for that broader durability.

---

## Scenario 06: hostPath Volume Type Mismatch

**Namespace:** `platform-ops`

**Context:** `node-inspector` is a diagnostic Pod that mounts a `hostPath` volume to read `/var/log` from the node it lands on. The pod won't even start — diagnosing it shows the event `/var/log is not a file`.

**Concept:** `hostPath.type` tells Kubernetes what it should find at that path on the node, and it checks — claim the wrong kind, and the Pod is rejected before it starts. Make sure `hostPath.type` matches what `/var/log` actually is on the node.

**Task:** `volumes` is immutable on an existing standalone Pod, so it can't just be edited in place. Correct the Pod manifest, then delete and replace the failed Pod.

**Issue found (two-part):**

1. `hostPath.type` was set to `File`, but `/var/log` on the node is a directory — Kubernetes validates this at admission and rejects the mismatch before the Pod starts.
2. After correcting the type, the container's `volumeMounts[].mountPath` was set to `/host-logs` instead of `/var/log`, so the volume was attached but exposed at the wrong path inside the container.

**Solution:** Export → edit → delete → reapply:

```yaml
volumes:
  - name: host-logs
    hostPath:
      path: /var/log
      type: Directory   # corrected from File
containers:
  - name: node-inspector
    volumeMounts:
      - name: host-logs
        mountPath: /var/log   # corrected from /host-logs
```

```bash
kubectl delete pod node-inspector -n platform-ops
kubectl apply -f node-inspector.yaml
```

---

## Scenario 07: Analytics Pod Blocked, Then Blocked Again

**Namespace:** `analytics-ops`

**Context:** Pod `report-writer` writes reports into PVC `analytics-writer`, using a `subPath` so its output lands in its own subdirectory instead of the PVC's root. The PVC requests `ReadWriteMany` storage, meant to bind to PV `analytics-pv`. An init container on the Pod already prepares the real directory structure (`reports/2024`) on the volume before the main container starts. This unfolds in two stages: first, PVC `analytics-writer` sits in `Pending` forever, and the Pod never gets past that. Once the PVC binds, a second problem appears: the Pod is stuck in `ContainerCreating`.

**Concept:**

1. A PVC only binds to a PV whose `accessModes` cover everything the PVC asks for — a PV missing even one requested mode is treated as no match at all.
2. `subPath` must point at a path that already exists inside the volume; Kubernetes won't create missing folders for you at mount time, so a typo'd path fails the mount outright even if the correct folder exists one character away.

**Task:**

1. Compare the PV's `accessModes` against the PVC's request, and add the appropriate one to the list if needed.
2. Once the PVC shows `Bound`, check the Pod's `volumeMounts[].subPath` value — make sure it matches the folder the init container actually created. Since `volumeMounts` is immutable on an existing Pod, delete and recreate the Pod after fixing the manifest.

**Issue found (two-stage):**

1. PVC requested `ReadWriteMany`, but the PV only offered `ReadWriteOnce`.
2. The main container's `subPath` was `reports-2024` (hyphen) instead of `reports/2024` (slash), the exact path the init container created.

**Solution:**

```yaml
# PV fix
spec:
  accessModes:
    - ReadWriteOnce
    - ReadWriteMany   # added

# Pod fix (subPath corrected)
volumeMounts:
  - name: analytics-storage
    mountPath: /data
    subPath: reports/2024   # corrected from reports-2024
```

```bash
kubectl delete pod report-writer -n analytics-ops
kubectl apply -f report-writer.yaml
```

---

## Scenario 08: StatefulSet Pod Can't Find Its Own Volume

**Namespace:** `database-ops`

**Context:** `redis-cluster`, a StatefulSet, uses `volumeClaimTemplates` to give each replica its own dedicated storage volume, named `data`. The StatefulSet is not ready, and pods with label `app=redis-cluster` show nothing at all.

**Concept:** A StatefulSet's `volumeClaimTemplates[].metadata.name` is the name the controller uses to auto-generate a matching volume in every pod it creates. A container's `volumeMounts[].name` must reference that exact same name — get it wrong, and the Pod object the controller tries to create is invalid, so the API server rejects it outright before a pod ever exists.

**Task:** Investigate StatefulSet `redis-cluster`, compare the `volumeClaimTemplates[].metadata.name` against the container's `volumeMounts[].name`. Fix the container's `volumeMounts` so it references the correct volume name. This field isn't immutable, so no delete/recreate is needed — just patch or reapply.

**Issue found:** The container's `volumeMounts[].name` was `cache`, not matching `volumeClaimTemplates[].metadata.name` (`data`).

**Solution:**

```bash
kubectl patch statefulset redis-cluster -n database-ops \
  --type='json' \
  -p='[{"op": "replace", "path": "/spec/template/spec/containers/0/volumeMounts/0/name", "value": "data"}]'
```

Verify:

```bash
kubectl get pods -n database-ops -l app=redis-cluster
kubectl get pvc -n database-ops
```

---

## Scenario 09: RBAC Role Missing Verb Causes Forbidden Error

**Namespace:** `pipelines-ops`

**Context:** CI service account `ci-deployer` is bound to Role `deployment-manager` via a RoleBinding. The Role's `rules` currently grant only `get`, `list`, and `watch` on `deployments` — enough to inspect and monitor. As its final step, the nightly pipeline is also supposed to restart the `api` Deployment so it picks up a freshly-updated ConfigMap. The pipeline fails, even though the same identity can successfully `get`/`list` Deployments.

**Concept:** RBAC is deny-by-default — a ServiceAccount can only do exactly what a Role/RoleBinding explicitly lists, verb by verb. Being allowed to view a resource says nothing about being allowed to modify it.

**Task:** Inspect Role `deployment-manager`'s `rules` and update them as needed for the pipeline's action.

**Issue found:** Only `get`, `list`, `watch` were granted; restarting a Deployment requires `patch`.

**Solution:**

```bash
kubectl patch role deployment-manager -n pipelines-ops \
  --type='json' \
  -p='[{"op": "replace", "path": "/rules/0/verbs", "value": ["get", "list", "watch", "patch"]}]'
```

Verify:

```bash
kubectl auth can-i patch deployments \
  --as=system:serviceaccount:pipelines-ops:ci-deployer -n pipelines-ops
```

---

## Scenario 10: KubeConfig Pointed at the Wrong Cluster, Then the Wrong User

**Context:** A separate kubeconfig file is at `~/practice-kubeconfig`, with two contexts: `cluster-old` and `cluster-prod`. The team decommissioned `cluster-old` last week, and a new production cluster, `cluster-prod`, was recently created together with a new user, `engineer-admin`, who has write access.

**Concept:** A kubeconfig "context" is just a named pairing of (which cluster + which user credentials). `kubectl` always operates against whichever context is `current-context` — switching clusters and switching identities are two independent settings, and either one can be wrong without the other being wrong too. Reads working while writes fail is a strong signal you've got the right cluster but the wrong identity, not a connectivity problem.

**Task (Part 1):** Running `kubectl` commands failed with `Unable to connect to the server: dial tcp: lookup cluster-old-api ... no such host`. Make sure you're in the correct context pointing at the real production cluster, then show the current context and save the output to `current-context.txt`.

**Task (Part 2):** After fixing the context, `kubectl get pods -A` worked, but `kubectl apply -f new-deployment.yaml` failed with a Forbidden error from `contractor-readonly`. Run `kubectl config view`, confirm the real prod cluster's `user` field points at the identity with correct access, and fix it with `kubectl config set-context cluster-prod --user=<username>`.

**Issue found (two-part):**

1. Active context was `cluster-old`, pointing at a decommissioned cluster.
2. After switching to `cluster-prod`, its bound user was `contractor-readonly` (read-only), not `engineer-admin`.

**Solution:**

```bash
export KUBECONFIG=~/practice-kubeconfig

# Fix cluster context
kubectl config use-context cluster-prod
kubectl config current-context > current-context.txt

# Fix user identity bound to the context
kubectl config set-context cluster-prod --user=engineer-admin
```

Verify:

```bash
kubectl config view --minify
kubectl apply -f new-deployment.yaml
```

---

## Scenario 11: ValidatingWebhook Rejecting Legitimate Pods

**Namespace:** `production-ops`

**Context:** A `ValidatingWebhookConfiguration` called `enforce-labels` requires every Pod to carry a `team` label, enforced by webhook service `label-validator`. A routine rollout of Deployment `checkout-service` fails with `admission webhook "enforce-labels.production-ops.svc" denied the request: missing required label "team"` — even though `team: retail` is clearly visible on the Deployment's own `metadata.labels`.

**Concept:** A Deployment doesn't run anything itself — it creates Pods for you. The webhook checks the Pod, not the Deployment, and labels on the Deployment's own `metadata` don't automatically carry over to the Pods it creates. Only labels under `spec.template.metadata.labels` get copied onto every Pod.

**Task:** Check the Pod template's labels in the Deployment's manifest, and make sure the Pods it creates actually carry the label the webhook is checking for.

**Issue found:** `team: retail` existed only on the Deployment's top-level metadata, not on `spec.template.metadata.labels`.

**Solution:**

```yaml
spec:
  template:
    metadata:
      labels:
        app: checkout
        team: retail   # added to the Pod template specifically
```

Verify:

```bash
kubectl get pods -n production-ops -l app=checkout --show-labels
```

---

## Scenario 12: Deprecated API Version Rejected by API Server

**Namespace:** `web-ops`

**Context:** `legacy-ingress.yaml` is a manifest for an Ingress named `legacy-ingress`, written years ago against an old Kubernetes API. The cluster has since been upgraded, and nobody has re-applied this manifest since.

**Concept:** Kubernetes periodically retires old API versions in favor of a newer, stable one — and the shape of the fields often changes too, not just the version string.

**Task:** Update the manifest's `apiVersion` to the correct one, and update the backend field structure to match the current schema (`spec.rules[].http.paths[].backend.service.name` / `port.number`) before reapplying. `networking.k8s.io/v1` also requires a `pathType` on every path entry (no default value).

**Issue found:** Manifest used the removed `extensions/v1beta1` API with the old flat `serviceName`/`servicePort` backend fields.

**Solution:**

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: legacy-ingress
  namespace: web-ops
spec:
  rules:
  - host: docs.example.com
    http:
      paths:
      - path: /
        pathType: Prefix   # required in v1, no default
        backend:
          service:
            name: legacy-docs-svc
            port:
              number: 80
```

```bash
kubectl apply -f legacy-ingress.yaml
```

---

## Scenario 13: Custom Resource Rejected by CRD Schema Validation

**Namespace:** `tools-ops`

**Context:** CRD `databases.ops.example.com` requires field `spec.engine` to be one of `postgres` or `mysql`. A new custom resource, `orders-db`, was just submitted for the database operator to provision — meant to be a Postgres database for the order-tracking service.

**Concept:** A CustomResourceDefinition (CRD) lets a cluster define its own resource type with its own schema, which the API server validates at submission time.

**Task:** Check the `orders-db` Database resource's spec against the CRD's schema to find any issues, so it passes CRD schema validation and the operator can pick it up (`orders-db` is meant to be Postgres — fix the typo accordingly). Apply changes.

**Issue found:** `spec.engine: postgress` — a typo that doesn't exact-match the CRD's enum.

**Solution:**

```yaml
spec:
  engine: postgres   # corrected from "postgress"
```

```bash
kubectl apply -f orders-db.yaml
```

---

## Scenario 14: Helm Install Failing Because the Chart Repo Was Never Added

**Namespace:** `tools-ops`

**Context:** The onboarding checklist says to deploy the `podinfo` chart with `helm install internal-api opscorp/podinfo -n tools-ops --create-namespace`. Every other engineer already has this working. The install fails with `Error: INSTALLATION FAILED: failed to download "opscorp/podinfo" (hint: running helm repo update may help)`.

**Concept:** Helm repos are configured per machine, not per cluster — `helm repo add` is a one-time local setup step. A brand-new laptop has none configured at all, even if the exact same install already works for everyone else on their own machines.

**Task:** Check if the chart repo is in the list. If not, add repo `opscorp https://stefanprodan.github.io/podinfo`, update, and install.

**Verify:** `helm list -n tools-ops` should show `internal-api` with `STATUS: deployed`. To confirm it's actually serving traffic, port-forward to it and confirm a JSON response starting with `"message": "greetings from podinfo"`.

**Issue found:** No Helm repos configured on this machine at all.

**Solution:**

```bash
helm repo add opscorp https://stefanprodan.github.io/podinfo
helm repo update
helm install internal-api opscorp/podinfo -n tools-ops --create-namespace
```

Verify:

```bash
helm list -n tools-ops
kubectl -n tools-ops port-forward --address 0.0.0.0 deploy/internal-api-podinfo 9898:9898
curl http://<host-or-public-ip>:9898
```

---

## Scenario 15: Kustomize Can't Find One of Its Resource Files

**Context:** `base/kustomization.yaml` lists two files under `resources:` — the Deployment and Service manifests that make up the `web-app` application.

**Concept:** Kustomize's `resources:` field needs an exact filename match to the files actually on disk — there's no fuzzy matching. If even one listed file doesn't exist, Kustomize can't build its resource list at all, and the whole `apply -k` fails before creating anything.

**Task:** Run `ls base/` to see what files actually exist, and compare that against `base/kustomization.yaml`'s `resources:` list. Fix any kind of mismatch observed and reapply.

**Issue found:** A filename mismatch between what `resources:` listed and what actually existed on disk in `base/`.

**Solution:**

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
- deployment.yaml
- service.yaml
```

```bash
kubectl apply -k base
```

Verify:

```bash
kubectl get deployment,svc -n default
```

---

## Key Takeaways Across All Scenarios

|Concept|Scenarios|
|---|---|
|Services match Pod labels, not Deployment names|01|
|Silent defaults on omitted fields (`parallelism`)|02|
|NetworkPolicies are opt-in and deny-by-default once applied|03|
|No validation of referenced object names at apply-time|04, 13|
|Ephemeral container filesystem vs. volumes|05|
|`hostPath.type` and `mountPath` are independently configured|06|
|PV `accessModes` must be a superset of PVC's request|07|
|`subPath` requires the path to already exist|07|
|`volumeMounts.name` must match `volumeClaimTemplates.name` exactly|08|
|RBAC verbs are additive and independent (read ≠ write)|09|
|Kubeconfig context = cluster + user, independently wrong|10|
|Labels on Deployment metadata ≠ labels on Pod template|11|
|Deprecated API versions and changed field shapes|12|
|CRD schema validation (enums) is exact-match|13|
|Helm repos are local machine config, not cluster state|14|
|Kustomize `resources:` needs exact filename matches|15|