
Ingress Controller
# ==Ingress Controller — Object Summary (from diagram)==

The NGINX **Deployment** sits at the center, with 4 supporting objects:

| Object             | Name (from diagram)                | Purpose                                                                                                                                    |
| ------------------ | ---------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------ |
| **Deployment**     | `nginx-ingress-controller`         | Runs the actual NGINX controller pod(s) — 1 replica, listens on ports 80 (http) and 443 (https)                                            |
| **Service**        | `nginx-ingress` (type: `NodePort`) | Exposes the Deployment's pods externally — maps Service ports 80/443 → container ports 80/443, matched via `selector: name: nginx-ingress` |
| **ConfigMap**      | `nginx-configuration`              | Referenced in the Deployment's `args` via `--configmap=$(POD_NAMESPACE)/nginx-configuration` — holds controller config settings            |
| **ServiceAccount** | `nginx-ingress-serviceaccount`     | Identity for the controller pod, under "Auth" — connects to **Roles**, **ClusterRoles**, **RoleBindings** (RBAC) shown at the bottom right |

## How they connect (per the diagram's lines/grouping)

```
                    Deployment (nginx-ingress-controller)
                       │                    │
                  ConfigMap          ServiceAccount
              (nginx-configuration)  (nginx-ingress-serviceaccount)
                                            │
                                    Roles / ClusterRoles / RoleBindings
```

- **Deployment** connects downward to both **ConfigMap** and **ServiceAccount** (the two dotted lines from the NGINX icon).
- **Service** is shown separately (top right) — it selects the Deployment's pods via `selector: name: nginx-ingress`, and its `targetPort` values (80, 443) match the Deployment's `containerPort` values (80, 443).
- **ConfigMap** box (middle right) — just declares `kind: ConfigMap`, `name: nginx-configuration`, matching what the Deployment's `args` references.
- **Auth** box (bottom right) — `kind: ServiceAccount`, `name: nginx-ingress-serviceaccount`, tied to **Roles / ClusterRoles / RoleBindings** (RBAC permissions) shown as tabs underneath.

## Total objects shown in this diagram: 4

1. **Deployment** — runs the controller
2. **Service** — exposes the controller
3. **ConfigMap** — configures the controller
4. **ServiceAccount** (+ RBAC: Role/ClusterRole/RoleBinding) — authorizes the controller to talk to the API server
# Ingress Controller — Full Manifests (per diagram)

## 1. Deployment

```yaml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: nginx-ingress-controller
spec:
  replicas: 1
  selector:
    matchLabels:
      name: nginx-ingress
  template:
    metadata:
      labels:
        name: nginx-ingress
    spec:
      containers:
        - name: nginx-ingress-controller
          image: quay.io/kubernetes-ingress-controller/nginx-ingress-controller:0.21.0
          args:
            - /nginx-ingress-controller
            - --configmap=$(POD_NAMESPACE)/nginx-configuration
          env:
            - name: POD_NAME
              valueFrom:
                fieldRef:
                  fieldPath: metadata.name
            - name: POD_NAMESPACE
              valueFrom:
                fieldRef:
                  fieldPath: metadata.namespace
          ports:
            - name: http
              containerPort: 80
            - name: https
              containerPort: 443
```

## 2. Service

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-ingress
spec:
  type: NodePort
  ports:
    - port: 80
      targetPort: 80
      protocol: TCP
      name: http
    - port: 443
      targetPort: 443
      protocol: TCP
      name: https
  selector:
    name: nginx-ingress
```

## 3. ConfigMap

```yaml
kind: ConfigMap
apiVersion: v1
metadata:
  name: nginx-configuration
```

## 4. ServiceAccount (Auth)

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: nginx-ingress-serviceaccount
```

---

## How they reference each other

|Field|In|Points to|
|---|---|---|
|`spec.selector.matchLabels.name` (Deployment)|Deployment|Must match `template.metadata.labels.name` — self-reference for pod management|
|`spec.selector.name` (Service)|Service|`nginx-ingress` — matches the Deployment's pod label, so the Service routes to those pods|
|`spec.ports[].targetPort` (Service)|Service|`80`/`443` — matches `containerPort` in the Deployment's pod spec|
|`--configmap=$(POD_NAMESPACE)/nginx-configuration` (Deployment args)|Deployment|`metadata.name: nginx-configuration` in the ConfigMap|
|`env: POD_NAMESPACE` (Deployment, via Downward API)|Deployment|Resolves dynamically from `fieldPath: metadata.namespace`, used to build the `--configmap=` arg|

Note: the diagram doesn't show the full RBAC manifests (Role/ClusterRole/RoleBinding) — only the tab labels (`Roles`, `ClusterRoles`, `RoleBindings`) and the ServiceAccount itself. Want me to write out a typical Role/ClusterRole + RoleBinding pair that would grant this ServiceAccount permission to watch Ingress/Service/Secret/ConfigMap objects, so the set is complete?

# ==Ingress Resource — Explained==

## What it is

The **Ingress Resource** is the config file(s) (`kind: Ingress`) that tell the **Ingress Controller** how to route incoming traffic. The controller (NGINX) is the engine; the Resource is the rulebook it reads.

## The 4 rule scenarios (Image 4 & 5)

An Ingress Resource is built from **rules**, and each rule can handle one of these cases:

|Rule|Trigger|Example|
|---|---|---|
|**Rule 1**|Single backend, no host/path matching needed at all|`www.my-online-store.com` → one service, always|
|**Rule 2**|Same host, different **paths**|`wear.my-online-store.com/`, `/returns`, `/support` → all to the `wear` service|
|**Rule 3**|Different **host**, different paths|`watch.my-online-store.com/`, `/movies`, `/tv` → all to the `video`/`watch` service|
|**Rule 4**|Anything not matched by the above|Falls through to a default 404 backend|

This maps directly to two routing strategies:

- **Path-based routing** — same domain, different URL paths → different services (Rule 2/3)
- **Host-based routing** — different subdomains entirely → different services (splitting Rule 2 vs Rule 3)

## Manifest #1 — Simplest form: `defaultBackend` only (Image 2 & 3)

No rules at all — just catch everything and send it to one service. Used when you have **one single app**, no routing logic needed:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-wear
spec:
  defaultBackend:
    service:
      name: wear-service
      port: 80
```

```bash
kubectl create -f Ingress-wear.yaml
kubectl get ingress
# NAME           HOSTS   ADDRESS   PORTS   AGE
# ingress-wear   *       80        2s
```

## Manifest #2 — Path-based routing, single host (Image 6 & 9, left)

**1 Rule, 2 Paths** — same domain, split by path:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-wear-watch
spec:
  rules:
    - http:
        paths:
          - path: /wear
            backend:
              service:
                name: wear-service
                port: 80
          - path: /watch
            backend:
              service:
                name: watch-service
                port: 80
```

## Manifest #3 — Host-based routing, multiple hosts (Image 9, right)

**2 Rules** — different subdomains, each with its own path(s):

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-wear-watch
spec:
  rules:
    - host: wear.my-online-store.com
      http:
        paths:
          - path: /wear
            backend:
              service:
                name: wear-service
                port: 80
    - host: watch.my-online-store.com
      http:
        paths:
          - path: /watch
            backend:
              service:
                name: watch-service
                port: 80
```

**Key structural rule:** each `rules` entry = one **host**; each host can have multiple **paths**, and each path maps to one backend **Service**.

## Verifying it (Image 7)

```bash
kubectl describe ingress ingress-wear-watch
```

```
Rules:
  Host  Path    Backends
  ----  ----    --------
  *
        /wear   wear-service:80
        /watch  watch-service:80
Events:
  Normal  CREATE  14s  nginx-ingress-controller  Ingress default/ingress-wear-watch17
```

Note the **`Events` → `From: nginx-ingress-controller`** line — this is direct proof of the controller connection (explained below): the controller detected and processed this Ingress object itself.

## How it connects to the Ingress Controller

This is the mechanism tying everything together:

```
1. You create the Ingress Resource (kubectl create -f ...)
        ↓
2. This just writes an object into the Kubernetes API/etcd — nothing routes yet
        ↓
3. The Ingress Controller (NGINX pod) is continuously WATCHING the API
   for Ingress objects (via its ServiceAccount + RBAC permissions —
   covered earlier)
        ↓
4. Controller sees the new/updated Ingress Resource, reads its
   rules (host/path/backend), and reconfigures itself (generates
   an internal nginx.conf, reloads)
        ↓
5. From then on, incoming requests matching that host/path get
   proxied by NGINX to the specified backend Service
```

**In short:** the Ingress Resource never talks to Services directly — it's purely declarative data. The **Controller** is the active component that reads it and does the actual proxying. That's why `kubectl describe ingress` shows `From: nginx-ingress-controller` in its Events — it's a log entry generated _by the controller_ confirming it picked up and processed that Ingress object.

## One-line summary

> **Ingress Resource = the "what" (routing rules by host/path → Service). Ingress Controller = the "who" (the software that watches for those rules and enforces them).** Nothing routes until the controller reads the resource.





---

As we already discussed **Ingress** in our previous lecture. Here is an update.

In this article, we will see what changes have been made in previous and current versions of **Ingress**.

Like in **apiVersion**, **serviceName** and **servicePort** etc.

[_![](https://kodekloud.com/kk-media/image/upload/v1698321206/1200736109541070.InaagGGYE8f31Jm2PTKH_height640.png)_](https://kodekloud.com/kk-media/image/upload/v1702469282/course-resource-new/1200736109541070.InaagGGYE8f31Jm2PTKH_height640.png)

Now, in k8s version **1.20+,** we can create an Ingress resource in the imperative way like this:-

Format -

```
kubectl create ingress  --rule="host/path=service:port"
```

Example -

```
kubectl create ingress ingress-test --rule="wear.my-online-store.com/wear*=wear-service:80"
```

Find more information and examples in the below reference link:-

[https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands#-em-ingress-em-](https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands#-em-ingress-em-)

**References:-**

[https://kubernetes.io/docs/concepts/services-networking/ingress](https://kubernetes.io/docs/concepts/services-networking/ingress)

[https://kubernetes.io/docs/concepts/services-networking/ingress/#path-types](https://kubernetes.io/docs/concepts/services-networking/ingress/#path-types)







---


To display node IP address
### 1. Quick view — all nodes with internal/external IPs

```bash
kubectl get nodes -o wide
```
This gives you a table including `INTERNAL-IP` and `EXTERNAL-IP` columns alongside node name, status, roles, version, OS, etc.

### 2. Specific node, just the IPs
```bash
kubectl get node <node-name> -o jsonpath='{.status.addresses}'
```

Or cleaner, split by type:
```bash
kubectl get node <node-name> -o jsonpath='{range .status.addresses[*]}{.type}{"	"}{.address}{"
"}{end}'
```

Typical output:
```
InternalIP    10.0.1.15
ExternalIP    34.123.45.67
Hostname      node-1.internal
```
### 3. Full detail view
```bash
kubectl describe node <node-name>
```

Look for the `Addresses:` section near the top — shows `InternalIP`, `ExternalIP` (if any), and `Hostname`.




---
Different ingress controllers have different options that can be used to customise the way it works. NGINX Ingress controller has many options that can be seen [here](https://kubernetes.github.io/ingress-nginx/examples/). I would like to explain one such option that we will use in our labs. The [Rewrite](https://kubernetes.github.io/ingress-nginx/examples/rewrite/) target option.

Our `watch` app displays the video streaming webpage at `http://<watch-service>:<port>/`

Our `wear` app displays the apparel webpage at `http://<wear-service>:<port>/`

We must configure Ingress to achieve the below. When user visits the URL on the left, his/her request should be forwarded internally to the URL on the right. Note that the /watch and /wear URL path are what we configure on the ingress controller so we can forward users to the appropriate application in the backend. The applications don't have this URL/Path configured on them:

`http://<ingress-service>:<ingress-port>/watch` --> `http://<watch-service>:<port>/`

`http://<ingress-service>:<ingress-port>/wear` --> `http://<wear-service>:<port>/`

Without the `rewrite-target` option, this is what would happen:

`http://<ingress-service>:<ingress-port>/watch` --> `http://<watch-service>:<port>/watch`

`http://<ingress-service>:<ingress-port>/wear` --> `http://<wear-service>:<port>/wear`

Notice `watch` and `wear` at the end of the target URLs. The target applications are not configured with `/watch` or `/wear` paths. They are different applications built specifically for their purpose, so they don't expect `/watch` or `/wear` in the URLs. And as such the requests would fail and throw a `404` not found error.

To fix that we want to "ReWrite" the URL when the request is passed on to the watch or wear applications. We don't want to pass in the same path that user typed in. So we specify the `rewrite-target` option. This rewrites the URL by replacing whatever is under `rules->http->paths->path` which happens to be `/pay` in this case with the value in `rewrite-target`. This works just like a search and replace function.

For example: `replace(path, rewrite-target)`

In our case: `replace("/path","/")`

```
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: test-ingress
  namespace: critical-space
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
    - http:
        paths:
          - path: /pay
            pathType: Prefix
            backend:
              service:
                name: pay-service
                port:
                  number: 8282
```

In another example given [here](https://kubernetes.github.io/ingress-nginx/examples/rewrite/), this could also be:

`replace("/something(/|$)(.*)", "/$2")`

```
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: rewrite
  namespace: default
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /$2
spec:
  rules:
    - host: rewrite.bar.com
      http:
        paths:
          - path: /something(/|$)(.*)
            pathType: Prefix
            backend:
              service:
                name: http-svc
                port:
                  number: 80
```

Annotation manifest SAMPLE from kk-practice:
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: critical-ingress
  namespace: critical-space
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
    nginx.ingress.kubernetes.io/ssl-redirect: "false"
spec:
  rules:
  - http:
      paths:
      - backend:
          service:
            name: pay-service
            port:
              number: 8282
        path: /pay
        pathType: Prefix
status:
  loadBalancer: {}
```

annotations purpose:
# What These Two Annotations Do

## `nginx.ingress.kubernetes.io/rewrite-target: /`

This **rewrites the URL path** before forwarding the request to the backend Service — it strips off the matched Ingress path and replaces it with whatever you set (`/` in this case).

### Why it's needed

Going back to our `wear`/`watch` example:

```yaml
rules:
  - http:
      paths:
        - path: /wear
          backend:
            service:
              name: wear-service
```

Without the rewrite annotation, when a user hits `www.my-online-store.com/wear`, NGINX forwards the **full path** `/wear` to the `wear-service` backend. But your backend app is very likely built to respond at `/`, not `/wear` — it has no route defined for `/wear`, so it'll 404.

With the annotation set:

```yaml
metadata:
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
```

NGINX **strips `/wear`** from the path before proxying, so the backend receives a request at `/` instead. The backend app never even knows the `/wear` prefix existed — it just sees a normal request at its root.

### Visual

```
User requests:        www.my-online-store.com/wear/shoes
Ingress path match:    /wear
Rewrite target:         /
                        ↓
Backend receives:      /shoes      (the "/wear" prefix is stripped/rewritten)
```

This is essential any time your Ingress `path` is just being used as a **routing prefix** rather than something the backend app actually expects in its own URL scheme.

---

## `nginx.ingress.kubernetes.io/ssl-redirect: "false"`

This **disables automatic HTTP → HTTPS redirection** for this Ingress.

### Default behavior without this annotation

By default, if your Ingress has a `tls:` block configured (like the `secretName` we covered earlier), the NGINX controller **automatically redirects all plain HTTP requests to HTTPS** — e.g., someone hitting `http://my-online-store.com` gets a `301` redirect to `https://my-online-store.com`.

### Why you'd turn it off

You set `ssl-redirect: "false"` when:

- You **don't have TLS configured yet** for this Ingress (no `tls:` block / no cert) — without disabling it, some setups still error out or misbehave trying to force a redirect to HTTPS that doesn't actually work
- You're doing **local development/testing** and don't want to deal with certs at all
- **TLS termination happens elsewhere** (e.g., at an upstream load balancer or CDN in front of the Ingress Controller), so forcing another redirect at this layer is redundant or breaks that setup
- You genuinely want to **serve both HTTP and HTTPS** without forcing users onto HTTPS

### Note the quoted `"false"`

This is a common gotcha with NGINX Ingress annotations — the value must be a **string** `"false"`, not the YAML boolean `false`. NGINX ingress annotations are just plain strings under the hood (they get parsed by the controller, not by Kubernetes' schema), so `false` (unquoted) can behave unpredictably or get coerced incorrectly. Always quote it.

---

## Together, in context

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-wear
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
    nginx.ingress.kubernetes.io/ssl-redirect: "false"
spec:
  rules:
    - http:
        paths:
          - path: /wear
            backend:
              service:
                name: wear-service
                port: 80
```

This says: _"Route `/wear` traffic to `wear-service`, but strip the `/wear` prefix before forwarding (so the app sees `/`), and don't force HTTPS redirects since this Ingress isn't configured with TLS."_

## Tying back to earlier discussion

Remember from a few messages ago — we flagged that **annotations are controller-specific and not portable**. Both of these are perfect examples: they only work because you're using the **NGINX** Ingress Controller (`nginx.ingress.kubernetes.io/...` prefix). If you switched to Traefik or HAProxy, you'd need completely different annotation keys/syntax to achieve the same rewrite and SSL-redirect behavior — this is exactly the portability weakness of the base Ingress API that we noted earlier as part of why the Gateway API exists.