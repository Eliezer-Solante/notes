![[Pasted image 20260822185237.png]]
![[Pasted image 20260822185250.png]]

![[Pasted image 20260822185310.png]]
![[Pasted image 20260822185350.png]]

## Kustomize: Problem Statement and Ideology

### The Problem It Solves

When managing Kubernetes manifests across multiple environments (dev, staging, prod), teams traditionally faced a few bad options:

1. **Copy-paste manifests per environment** — leads to massive duplication and drift. A change to a common label or resource limit means editing N files.
2. **Templating engines (Helm)** — introduce a templating language (Go templates) on top of YAML, which means you're no longer looking at valid YAML — you're looking at text that _generates_ YAML. This adds a layer of abstraction, a templating syntax to learn, and potential for subtle bugs (e.g., broken indentation from a bad template variable).
3. **Sed/shell scripting over YAML** — fragile, not declarative, hard to reason about or version.

Kustomize's core problem statement: **How do you customize raw, vanilla Kubernetes YAML for different environments without templating, and without duplicating the entire manifest set?**

### The Ideology: Template-Free, Declarative Customization

Kustomize is built on a few opinionated principles:

- **"No templates, just YAML."** Every file Kustomize reads is a legitimate, valid Kubernetes manifest — not a template with `{{ }}` placeholders. This means you can always `kubectl apply -f` a raw manifest and reason about it independently, and tools/editors that understand YAML/K8s schemas work correctly on it.
    
- **Declarative patching, not imperative scripting.** Instead of writing a script that mutates YAML, you declare _what should be different_ (a patch, a new value, an added label) and Kustomize computes the merged result.
    
- **Composition over duplication (DRY).** You define a common baseline once and layer environment-specific differences on top, rather than repeating the whole manifest per environment.
    
- **Purity/reproducibility.** Kustomize has (historically) avoided things like conditionals, loops, or scripting logic that would make output non-deterministic or hard to statically reason about — favoring predictable, side-effect-free transformations (though it has gradually added features like generators and transformers with configurable behavior).
    
- **Native integration.** `kubectl` has built-in support (`kubectl apply -k`), and it's a standard part of the Kubernetes ecosystem (originally a SIG-cli subproject), so no extra binary/runtime is strictly required.
    

---

### Base and Overlays

This is the structural mechanism that implements the ideology above.

**Base**

- The base is the canonical, environment-agnostic set of manifests — Deployments, Services, ConfigMaps, etc. — plus a `kustomization.yaml` that lists which resources belong to it.
- It represents the "common denominator" — what's true across _all_ environments.

```
base/
├── kustomization.yaml
├── deployment.yaml
├── service.yaml
└── configmap.yaml
```

```yaml
# base/kustomization.yaml
resources:
  - deployment.yaml
  - service.yaml
  - configmap.yaml
```

**Overlays**

- An overlay references a base (or another overlay) and applies environment-specific _patches_ or _transformations_ on top — without touching the base files themselves.
- Overlays typically correspond to environments: `dev`, `staging`, `prod`.

```
overlays/
├── dev/
│   ├── kustomization.yaml
│   └── replica-patch.yaml
├── staging/
│   └── kustomization.yaml
└── prod/
    ├── kustomization.yaml
    └── resources-patch.yaml
```

```yaml
# overlays/prod/kustomization.yaml
resources:
  - ../../base

namePrefix: prod-
commonLabels:
  env: prod

patches:
  - path: resources-patch.yaml
    target:
      kind: Deployment
      name: my-app

replicas:
  - name: my-app
    count: 5
```

Building it:

```bash
kubectl kustomize overlays/prod/
# or
kubectl apply -k overlays/prod/
```

**Key properties of this model:**

|Aspect|Behavior|
|---|---|
|Base immutability|Base files are never edited per-environment; overlays only _reference and patch_|
|Layering|Overlays can stack (e.g., a "regional" overlay on top of "prod" overlay)|
|Merge strategy|Patches use strategic merge patch or JSON 6902 patch semantics — precise, targeted edits|
|Output|Always plain, valid Kubernetes YAML — inspectable and diffable before apply|

### Why This Matters in Practice

- **Auditability**: `kubectl kustomize overlays/prod` renders the _exact_ final YAML that will be applied — no hidden template logic to trace through.
- **Safety**: Because output is always concrete YAML, you can diff it against what's live in the cluster, or run it through policy/validation tools without needing a templating-aware linter.
- **GitOps-friendly**: Tools like ArgoCD and Flux natively support Kustomize because the "render" step is deterministic and side-effect-free — a huge deal for GitOps philosophy where the Git repo is the single source of truth.

The contrast with Helm is often the clearest way to see Kustomize's ideology: Helm treats configuration as a _program that generates YAML_ (parameterized, reusable across totally different apps via charts); Kustomize treats configuration as _YAML that gets progressively refined_ (patch-based, tightly coupled to a specific base). Neither is strictly better — Helm suits packaging/distributing reusable charts (e.g., installing Prometheus), while Kustomize suits managing _your own_ app's environment-specific variations without a packaging abstraction. Many real-world setups even combine them (`helm template | kustomize build` or `kustomize` overlays patching a rendered Helm chart).