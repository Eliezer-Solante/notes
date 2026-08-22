HELM
![[Pasted image 20260822185931.png]]
![[Pasted image 20260822190053.png]]
![[Pasted image 20260822190155.png]]



## Kustomize vs. Helm

Both solve the same underlying problem — managing Kubernetes manifests across environments/deployments — but from fundamentally different philosophies.

### Core Philosophical Difference
![[Pasted image 20260822190333.png]]
---

### Manifest Formats

**Kustomize** — every file is valid, standalone Kubernetes YAML. Nothing is templated.

```yaml
# deployment.yaml (in base/)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  replicas: 1
  template:
    spec:
      containers:
        - name: app
          image: my-app:1.0
```

Combined with a `kustomization.yaml` that lists resources and declares transformations (patches, name prefixes, labels, image tag overrides, config/secret generators):

```yaml
resources:
  - deployment.yaml
patches:
  - path: patch.yaml
```

There is no `{{ }}` syntax anywhere — you could `kubectl apply -f` any individual file and it would work.

**Helm** — manifests live inside a **chart**, and are Go templates, not valid YAML on their own:

```
mychart/
├── Chart.yaml
├── values.yaml
├── templates/
│   ├── deployment.yaml
│   └── service.yaml
└── charts/            # sub-charts/dependencies
```

```yaml
# templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Release.Name }}-app
spec:
  replicas: {{ .Values.replicaCount }}
  template:
    spec:
      containers:
        - name: app
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
```

```yaml
# values.yaml
replicaCount: 1
image:
  repository: my-app
  tag: "1.0"
```

You cannot `kubectl apply` this file directly — it must first be rendered (`helm template` or `helm install`) using a values file, which substitutes variables, evaluates conditionals/loops, and produces final YAML.

---

### Pros and Cons

**Kustomize**

Pros:

- No new language to learn — pure YAML in, YAML out; easy for anyone who knows Kubernetes already.
- Transparent and auditable — `kubectl kustomize .` shows you exactly what will be applied, with no hidden logic.
- Native to `kubectl` (`kubectl apply -k`) — no extra tooling/binary strictly required.
- Excellent for the "same app, different environment" use case — DRY without duplication.
- Deterministic/side-effect-free rendering, which GitOps tools (ArgoCD, Flux) favor heavily.
- Strategic merge and JSON patches are precise and Kubernetes-aware (they understand list merge keys, etc.).

Cons:

- Weak at _packaging and distribution_ — there's no first-class chart registry/versioning system comparable to Helm repos/OCI charts.
- No conditionals/loops by design — sometimes you genuinely need "if X then Y," and Kustomize's patch-based model gets awkward (workarounds: components, multiple overlays, generators).
- Less mature ecosystem of pre-built configs compared to Helm charts (most third-party software ships a Helm chart, not a Kustomize base).
- Patch conflicts can be fiddly to debug when overlays stack deeply.
- No built-in release/version lifecycle concept (no "helm rollback" equivalent) — that's left to your CI/CD or GitOps tool.

**Helm**

Pros:

- Massive ecosystem — most open-source Kubernetes software ships an official Helm chart (Prometheus, Grafana, Istio, cert-manager, ingress-nginx, etc.).
- True packaging/versioning model — charts have semantic versions, can be pushed to registries (including OCI-compliant ones), and easily shared.
- Powerful templating — conditionals, loops, functions, sub-charts/dependencies handle complex, highly parameterized deployments well.
- Built-in release management — `helm install/upgrade/rollback/uninstall` tracks release history and state in-cluster, enabling easy rollback.
- Good for one chart supporting many wildly different deployments (via `values.yaml` overrides) — it's a general-purpose parameterization engine, not tied to a specific base.

Cons:

- Templates aren't valid YAML — indentation/whitespace bugs in Go templates are a common, often frustrating failure mode.
- Less transparent — to know what will actually be deployed, you must render it (`helm template`) since the source templates alone don't tell you the final manifest.
- Steeper learning curve — Go template syntax, sprig functions, `.Values`/`.Release`/`.Chart` objects are all Helm-specific knowledge.
- Historically had security/state concerns with Tiller (Helm v2) — resolved in Helm v3 (no more server-side Tiller), but worth noting for legacy contexts.
- Values files can get deeply nested and hard to trace for very complex charts (a value on line 200 might feed six different templated fields).
- Rendering is technically Turing-complete-ish (loops/conditionals), which trades some of the "predictability" that pure-YAML tools offer.

---

### Convergence in Practice

The tools aren't mutually exclusive, and a lot of real pipelines use both:

- **`helm template | kustomize build -`** — pull in a third-party Helm chart, render it to plain YAML, then apply Kustomize overlays/patches for environment-specific tweaks, without forking the chart.
- **Kustomize for your own apps, Helm for third-party dependencies** — a very common split: your team's microservices use Kustomize overlays (dev/staging/prod), while cluster add-ons (ingress controllers, monitoring stack) are installed via Helm charts maintained by their upstream projects.
- **ArgoCD/Flux support both natively** — so the GitOps layer doesn't force a choice; you pick per-application based on which model fits better.

### Bottom Line

- Choose **Kustomize** when you own the manifests, want zero templating overhead, and need environment-specific variants of a fixed app.
- Choose **Helm** when you're consuming or distributing a reusable, parameterized package — especially third-party software — and want built-in release/version management.