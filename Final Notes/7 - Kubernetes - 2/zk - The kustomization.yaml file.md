![[Pasted image 20260822191149.png]]

![[Pasted image 20260822191329.png]]
NOTE: using `kustomize build k8s/` does not apply/deploy any configs (does not create objects)
![[Pasted image 20260822191356.png]]

## The `kustomization.yaml` File

This is the heart of Kustomize — every directory that Kustomize operates on must contain a `kustomization.yaml` (or `Kustomization`) file. It's a declarative manifest that tells Kustomize _what resources to include_ and _what transformations to apply_ to them. It is itself a Kubernetes-style YAML object (`apiVersion: kustomize.config.k8s.io/v1beta1`, `kind: Kustomization`), even though it's not something you'd `kubectl apply` directly — it's consumed by the `kustomize`/`kubectl -k` build process.

### Basic Structure

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - deployment.yaml
  - service.yaml
  - configmap.yaml

namePrefix: prod-
namespace: production

commonLabels:
  app: my-app
  env: production

commonAnnotations:
  team: platform

images:
  - name: my-app
    newTag: "2.1.0"

replicas:
  - name: my-app
    count: 3
```

### Key Fields, Grouped by Purpose

**1. Bringing resources in**

- `resources` — list of plain Kubernetes manifest files, or paths to other directories containing their own `kustomization.yaml` (this is how a base gets referenced from an overlay).
    
    ```yaml
    resources:  - deployment.yaml  - ../../base          # pulling in a base from an overlay
    ```
    
- `components` — reusable, optional units of configuration that can be mixed into multiple overlays (introduced to solve composition problems `resources` couldn't handle cleanly).
- `crds` — for registering CustomResourceDefinition schemas so Kustomize can correctly patch custom resources.

**2. Generating resources**

- `configMapGenerator` — generates a ConfigMap from literals or files, and (importantly) auto-appends a content hash to its name so Pods automatically roll when the ConfigMap changes.
    
    ```yaml
    configMapGenerator:  - name: app-config    literals:      - LOG_LEVEL=debug    files:      - app.properties
    ```
    
- `secretGenerator` — same idea, for Secrets.
    
    ```yaml
    secretGenerator:  - name: db-creds    literals:      - password=supersecret
    ```
    
- `generatorOptions` — controls behavior of the generators above (e.g. disabling the name-hash suffix).

**3. Patching/transforming resources**

- `patches` — the general-purpose patch mechanism; supports both strategic merge patches and JSON 6902 patches, optionally scoped to a `target` selector (kind/name/labelSelector).
    
    ```yaml
    patches:  - path: increase-replicas.yaml    target:      kind: Deployment      name: my-app
    ```
    
- `patchesStrategicMerge` (legacy/older syntax) — same idea, strategic merge only.
- `patchesJson6902` (legacy/older syntax) — same idea, JSON patch only.
- `replicas` — shorthand specifically for overriding replica counts without writing a full patch.
- `images` — shorthand for overriding image name/tag/digest without a patch — very commonly used for CI/CD pipelines injecting a new tag.
    
    ```yaml
    images:  - name: my-app    newName: myregistry/my-app    newTag: sha-abc123
    ```
    

**4. Metadata-wide transformations**

- `namePrefix` / `nameSuffix` — prepend/append a string to every resource's `metadata.name`.
- `namespace` — force all resources into a specific namespace, overriding whatever they had (or lack).
- `commonLabels` — adds labels to `metadata.labels` _and_ to relevant label selectors (Service selectors, Deployment `matchLabels`, etc.) — this is important: it's selector-aware, not just a blind label stamp.
- `commonAnnotations` — adds annotations to all resources' metadata only (not selectors, since annotations aren't used for selection).

**5. Ordering / behavior control**

- `bases` — deprecated alias for referencing another kustomization directory via `resources` (older Kustomize versions used this separately).
- `vars` (deprecated in favor of replacements) — used to reference a field's value in one resource and substitute it elsewhere.
- `replacements` — the modern, more explicit successor to `vars`; copies a value from a source field into one or more target fields.
    
    ```yaml
    replacements:  - source:      kind: ConfigMap      name: app-config      fieldPath: data.API_URL    targets:      - select:          kind: Deployment        fieldPaths:          - spec.template.spec.containers.[name=app].env.[name=API_URL].value
    ```
    

### A Typical Base/Overlay Pair

```yaml
# base/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - deployment.yaml
  - service.yaml
```

```yaml
# overlays/prod/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - ../../base

namePrefix: prod-
namespace: production
commonLabels:
  env: prod

replicas:
  - name: my-app
    count: 5

images:
  - name: my-app
    newTag: "2.1.0"

patches:
  - path: resource-limits-patch.yaml
```

Running `kubectl kustomize overlays/prod/` walks this file, pulls in the base, applies every transformation in order, and prints the final, plain Kubernetes YAML — nothing hidden, nothing templated.

### A Few Practical Notes

- **Ordering matters conceptually but Kustomize tries to be declarative about it** — generators run, then transformers (patches, prefixes, labels) apply, roughly in a fixed pipeline rather than strictly the order you list fields in the file.
- **Field deprecations happen over versions** — `bases`, `patchesStrategicMerge`, `patchesJson6902`, and `vars` are all considered legacy in favor of the unified `resources`, `patches`, and `replacements` fields respectively. Worth checking `kustomize version` and the [official docs](https://kubectl.docs.kubernetes.io/references/kustomize/kustomization/) since the schema has evolved.
- **The file is the sole "entry point"** — you never invoke Kustomize by pointing at individual YAML files; you always point it at a directory containing a `kustomization.yaml`, which then declares everything else.