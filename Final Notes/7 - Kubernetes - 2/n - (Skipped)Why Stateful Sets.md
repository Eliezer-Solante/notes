### StatefulSets — what they are

**StatefulSet** is a Kubernetes workload API object for managing **stateful applications** that require stable, unique network IDs and persistent storage across pod restarts and rescheduling. Unlike Deployments (which treat pods as interchangeable), StatefulSets give each pod a stable identity and predictable lifecycle so the application can rely on ordered startup, stable hostnames, and durable volumes.

### Why use StatefulSets (key problems they solve)

|**Attribute**|**Why it matters**|
|---|---|
|**Stable network identity**|Each pod gets a predictable DNS name (pod-ordinal) so cluster members can find each other reliably.|
|**Stable storage**|PersistentVolumeClaims (PVCs) are created per pod and remain bound to that pod’s identity across restarts.|
|**Ordered, graceful scaling and updates**|Pods are created, deleted, and updated in a controlled sequence (by ordinal), which helps with leader election, quorum formation, and safe rolling upgrades.|
|**One-to-one mapping**|Useful for databases, clustered caches, and other systems that require a fixed identity per replica.|

### Headless Services — role and behavior

**Headless Service** is a Service with `clusterIP: None`. It does not provide load-balancing or a single virtual IP. Instead it enables direct DNS records for each pod backing the service.

#### Why headless services are used with StatefulSets

- **Per-pod DNS entries**: Kubernetes creates DNS A records like `pod-0.my-svc.namespace.svc.cluster.local`, `pod-1...`, enabling clients and peers to address a specific replica.
    
- **Service discovery for clustered apps**: Cluster members can discover each other and form a cluster using stable hostnames.
    
- **No proxy/load-balancer**: Traffic can be routed directly to a chosen pod rather than round-robin through a service proxy.
    

### Storage in StatefulSets

StatefulSets integrate with Kubernetes storage primitives to give each pod durable storage.

#### PersistentVolumeClaims per pod

- When you define a `volumeClaimTemplates` in a StatefulSet, Kubernetes creates a **PVC for each pod** (e.g., `data-my-statefulset-0`, `data-my-statefulset-1`).
    
- Each PVC is bound to a PersistentVolume (PV) and remains associated with the pod identity even if the pod is deleted and recreated on another node.
    

#### Storage class and reclaim policy

- PVCs use a **StorageClass** to provision PVs dynamically (if supported by the cluster).
    
- The **reclaim policy** of the PV (e.g., `Retain`, `Delete`) controls whether the underlying storage is preserved after the PVC is deleted. For stateful apps you often prefer `Retain` to avoid accidental data loss.
    

#### Access modes and volume types

- Choose volume types and access modes based on your app:
    
    - **ReadWriteOnce (RWO)** is common for single-writer databases.
        
    - **ReadWriteMany (RWX)** may be used for shared file systems (NFS, CSI drivers that support RWX).
        
- Some clustered databases require shared storage or application-level replication rather than shared volumes.
    

#### Pod deletion and PVC lifecycle

- Deleting a pod does **not** delete its PVC by default; PVCs persist until explicitly removed.
    
- When scaling down, StatefulSet will remove the pod but leave the PVC unless you delete the PVC or the StatefulSet’s owner references cause cleanup.
    

### Pod identity and DNS naming scheme

- Pod names follow the pattern: `<statefulset-name>-<ordinal>` (e.g., `web-0`, `web-1`).
    
- With a headless service `svc`, DNS entries become: `<pod>.<svc>.<namespace>.svc.cluster.local`.
    
- This deterministic naming is essential for bootstrapping clusters (e.g., telling replica 0 to be the initial seed).
    

### Lifecycle ordering and update strategy

- **OrderedReady** (default): Pods are created and become Ready in ordinal order; deletions and updates also proceed in reverse ordinal order. This ensures safe bootstrapping and teardown.
    
- **OnDelete**: Pods are only recreated when manually deleted, giving operators full control.
    
- **Rolling updates**: StatefulSet supports rolling updates but preserves identity and storage; you can control partitioning to update subsets safely.
    

### Typical use cases

- Databases: **MySQL, PostgreSQL, Cassandra, MongoDB** (where each node has identity and persistent data).
    
- Distributed systems requiring stable identity: **ZooKeeper, etcd, Kafka (brokers)**.
    
- Any application that needs stable network IDs and persistent per-replica storage.
    

### Best practices and operational notes

- **Use headless Service** for discovery; use a separate ClusterIP Service if you need load-balanced access.
    
- **Prefer dynamic provisioning** with a StorageClass that matches performance and durability needs.
    
- **Set appropriate reclaim policy** (often `Retain`) to avoid accidental data loss.
    
- **Plan for backups and restores** at the application level; PVCs and PVs are storage primitives, not backups.
    
- **Avoid shared RWX volumes** for stateful databases unless the database supports multi-writer semantics.
    
- **Test scaling and failover** behavior in a staging environment to validate cluster bootstrapping and recovery.
    
- **Control updates** with partitions or `OnDelete` when upgrading critical stateful clusters.
    

### Short example (conceptual YAML pieces)

- **Headless Service**
    

yaml

```
apiVersion: v1
kind: Service
metadata:
  name: my-svc
spec:
  clusterIP: None
  selector:
    app: my-stateful
  ports:
  - port: 5432
```

- **StatefulSet with volumeClaimTemplates**
    

yaml

```
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: my-stateful
spec:
  serviceName: "my-svc"
  replicas: 3
  selector:
    matchLabels:
      app: my-stateful
  template:
    metadata:
      labels:
        app: my-stateful
    spec:
      containers:
      - name: db
        image: my-db-image
        ports:
        - containerPort: 5432
        volumeMounts:
        - name: data
          mountPath: /var/lib/data
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: fast-ssd
      resources:
        requests:
          storage: 10Gi
```

### Quick troubleshooting checklist

- Pods not getting stable DNS: confirm headless Service `clusterIP: None` and `serviceName` in StatefulSet match.
    
- PVCs not created: check `volumeClaimTemplates` syntax and StorageClass availability.
    
- Data lost after pod deletion: inspect PV reclaim policy and whether PVCs were deleted.
    
- Cluster not forming: verify pod ordinals, readiness probes, and ordered startup behavior.
    

If you want, I can:

- Walk through a concrete example for a specific database (e.g., PostgreSQL or Cassandra) and show how to bootstrap and recover a cluster.
    
- Explain how CSI drivers and dynamic provisioning interact with StatefulSets in your cloud or on-prem environment.
    
- Help design a backup/restore strategy for StatefulSet-backed storage.