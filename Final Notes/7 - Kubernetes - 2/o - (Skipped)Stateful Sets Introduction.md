Here's a complete StatefulSet manifest with every section explained — StatefulSets are used when Pods need **stable identity, stable storage, and ordered deployment/scaling** (databases, Kafka, Zookeeper, Elasticsearch, etc.) — unlike a Deployment, where Pods are interchangeable.

## Full manifest

```yaml
apiVersion: v1
kind: Service
metadata:
  name: mysql-headless
  labels:
    app: mysql
spec:
  clusterIP: None          # headless service — required for StatefulSets
  selector:
    app: mysql
  ports:
    - port: 3306
      name: mysql
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql
spec:
  serviceName: mysql-headless    # must match the headless Service above
  replicas: 3
  selector:
    matchLabels:
      app: mysql
  podManagementPolicy: OrderedReady   # default; can be "Parallel"
  updateStrategy:
    type: RollingUpdate
    rollingUpdate:
      partition: 0
  template:
    metadata:
      labels:
        app: mysql
    spec:
      terminationGracePeriodSeconds: 30
      containers:
        - name: mysql
          image: mysql:8.0
          ports:
            - containerPort: 3306
              name: mysql
          env:
            - name: MYSQL_ROOT_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: mysql-secret
                  key: root-password
          volumeMounts:
            - name: data
              mountPath: /var/lib/mysql
          resources:
            requests:
              cpu: "500m"
              memory: "1Gi"
            limits:
              cpu: "1"
              memory: "2Gi"
  volumeClaimTemplates:
    - metadata:
        name: data
      spec:
        accessModes: ["ReadWriteOnce"]
        storageClassName: gp3
        resources:
          requests:
            storage: 10Gi
```

## Section-by-section breakdown

### 1. The headless Service (`clusterIP: None`)

```yaml
spec:
  clusterIP: None
```

**Why it's required:** A StatefulSet needs a way to give each Pod its **own stable DNS name**, not just one shared load-balanced IP. A headless Service does exactly that — instead of load-balancing, DNS returns the individual Pod IPs directly.

This is what enables each Pod to be reached at:

```
mysql-0.mysql-headless.default.svc.cluster.local
mysql-1.mysql-headless.default.svc.cluster.local
mysql-2.mysql-headless.default.svc.cluster.local
```

### 2. `spec.serviceName`

```yaml
serviceName: mysql-headless
```

Links the StatefulSet to the headless Service above — this is mandatory and is how Kubernetes wires up the per-Pod DNS records.

### 3. `spec.selector` / Pod labels

Same as a Deployment — tells the StatefulSet which Pods it owns, based on matching labels.

### 4. `podManagementPolicy`

```yaml
podManagementPolicy: OrderedReady   # or: Parallel
```

- **`OrderedReady`** (default) — Pods are created/scaled/terminated **one at a time, in order** (`mysql-0` must be Running/Ready before `mysql-1` starts). Used when Pods depend on each other coming up in sequence (e.g., a primary must exist before replicas join).
- **`Parallel`** — all Pods are created/deleted simultaneously, no ordering guarantees. Use when Pods are independent of each other's startup order.

### 5. `updateStrategy`

```yaml
updateStrategy:
  type: RollingUpdate
  rollingUpdate:
    partition: 0
```

Controls how updates roll out:

- **`RollingUpdate`** (default) — Pods are updated **in reverse ordinal order** (highest number first: `mysql-2` → `mysql-1` → `mysql-0`).
- **`partition`** — Pods with an ordinal **less than** this number are _not_ updated. Useful for canary-style rollouts — e.g., set `partition: 2` to only update `mysql-2`, leaving `mysql-0` and `mysql-1` untouched until you're confident.
- **`OnDelete`** (alternative type) — Pods are only updated when you manually delete them; no automatic rollout at all.

### 6. `template` (Pod spec)

Same structure as a Deployment's Pod template — containers, env vars, ports, resources. The key difference is _how_ these Pods get created (ordered, named predictably) rather than what's inside them.

### 7. `volumeClaimTemplates` — the defining feature

```yaml
volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: gp3
      resources:
        requests:
          storage: 10Gi
```

**This is the core thing that separates StatefulSets from Deployments.** Instead of one shared volume, Kubernetes automatically creates a **separate PersistentVolumeClaim per Pod**, each named predictably:

```
data-mysql-0
data-mysql-1
data-mysql-2
```

**Critically:** if `mysql-1` crashes and gets rescheduled, it comes back and **reattaches to the same PVC** (`data-mysql-1`) — it doesn't get a fresh empty volume like a Deployment's Pod would. This is what gives StatefulSet Pods **persistent, sticky identity and storage**.

## Deployment vs. StatefulSet — quick comparison

||Deployment|StatefulSet|
|---|---|---|
|Pod naming|Random suffix (`nginx-56cd7bd595-8lqdb`)|Predictable, ordinal (`mysql-0`, `mysql-1`, `mysql-2`)|
|Pod identity across restarts|New identity each time|**Same identity** preserved (name + storage)|
|Storage|Shared or none — Pods are interchangeable|**Dedicated PVC per Pod**, sticky across restarts|
|Startup/scaling order|No ordering — all Pods equal|**Ordered** (0, 1, 2... by default)|
|Needs headless Service?|No|**Yes** — required for stable per-Pod DNS|
|Typical use case|Stateless web apps, APIs|Databases, message queues, anything needing stable identity/storage (MySQL, Kafka, Cassandra, Zookeeper, Elasticsearch)|

## When you'd actually reach for a StatefulSet

- **Databases** replicated across Pods (each replica needs its own persistent disk, and often a primary/replica startup order matters)
- **Distributed systems** where each node needs a stable identity to find its peers (Kafka brokers, Zookeeper ensemble, Cassandra ring, etcd cluster)
- Anything where **"Pod comes back with the same name and same data"** matters — a Deployment gives you neither of those guarantees.