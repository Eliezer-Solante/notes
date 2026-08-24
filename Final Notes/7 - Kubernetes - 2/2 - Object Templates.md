# Kubernetes Object Templates Reference

For each object: a **Simple** minimal working template (no comments), and a **Full** template showing (nearly) all commonly-used fields — commented with what each section/field is _for_, and enum-style fields also show their allowed values.

---

## 1. Pod

### Simple

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
spec:
  containers:
    - name: app
      image: nginx:latest
```

### Full

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
  namespace: default               # which namespace this object lives in
  labels:                          # identifying key/values used by selectors (Services, Deployments, etc.)
    app: my-app
    tier: frontend
  annotations:                     # non-identifying metadata, for tools/humans, not used for selection
    description: "example pod"
spec:
  restartPolicy: Always            # what to do when a container exits — Always | OnFailure | Never
  terminationGracePeriodSeconds: 30  # how long to wait for graceful shutdown before force-killing
  serviceAccountName: my-sa        # identity the pod uses to talk to the API server
  nodeSelector:                    # simple key/value constraint on which node this can be scheduled to
    disktype: ssd
  affinity:                        # richer scheduling rules than nodeSelector
    nodeAffinity:                  # attract/require pod onto nodes matching label rules
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
          - matchExpressions:
              - key: kubernetes.io/os
                operator: In
                values: ["linux"]
    podAntiAffinity:                # keep this pod away from other pods matching a selector (e.g. spread replicas)
      preferredDuringSchedulingIgnoredDuringExecution:
        - weight: 100
          podAffinityTerm:
            labelSelector:
              matchLabels:
                app: my-app
            topologyKey: kubernetes.io/hostname
  tolerations:                     # lets this pod be scheduled onto nodes with matching taints
    - key: "key1"
      operator: "Equal"
      value: "value1"
      effect: "NoSchedule"
  topologySpreadConstraints:       # spreads pods evenly across zones/nodes/etc for HA
    - maxSkew: 1
      topologyKey: topology.kubernetes.io/zone
      whenUnsatisfiable: DoNotSchedule   # DoNotSchedule | ScheduleAnyway
      labelSelector:
        matchLabels:
          app: my-app
  initContainers:                  # run to completion, in order, before the main containers start (setup tasks)
    - name: init-setup
      image: busybox:latest
      command: ["sh", "-c", "echo init"]
  containers:                      # the actual application container(s) that run for the pod's lifetime
    - name: app
      image: nginx:1.27
      imagePullPolicy: IfNotPresent   # when to pull the image — Always | IfNotPresent | Never
      command: ["/bin/sh"]          # overrides the image's ENTRYPOINT
      args: ["-c", "nginx -g 'daemon off;'"]   # overrides the image's CMD (or args to command)
      workingDir: /app             # working directory inside the container
      ports:                       # informational: which ports the container listens on
        - name: http
          containerPort: 80
          protocol: TCP            # TCP | UDP | SCTP
      env:                         # environment variables set inside the container
        - name: ENV_VAR
          value: "value1"          # a literal value
        - name: FROM_SECRET
          valueFrom:
            secretKeyRef:          # pulls a single value out of a Secret
              name: my-secret
              key: password
        - name: FROM_CONFIGMAP
          valueFrom:
            configMapKeyRef:       # pulls a single value out of a ConfigMap
              name: my-config
              key: setting
        - name: POD_NAME
          valueFrom:
            fieldRef:               # injects pod's own metadata (Downward API)
              fieldPath: metadata.name
      envFrom:                     # bulk-import every key from a ConfigMap/Secret as env vars
        - configMapRef:
            name: my-config
        - secretRef:
            name: my-secret
      resources:                   # CPU/memory the scheduler reserves and the kubelet enforces
        requests:                  # guaranteed minimum, used for scheduling decisions
          cpu: "250m"
          memory: "128Mi"
        limits:                    # hard ceiling, container is throttled/killed if exceeded
          cpu: "500m"
          memory: "256Mi"
      volumeMounts:                # where volumes defined below get mounted inside this container
        - name: config-vol
          mountPath: /etc/config
        - name: data-vol
          mountPath: /data
      livenessProbe:               # restarts the container if this check fails (is it alive?)
        httpGet:
          path: /healthz
          port: 80
        initialDelaySeconds: 10
        periodSeconds: 10
        timeoutSeconds: 1
        failureThreshold: 3
      readinessProbe:              # removes pod from Service endpoints if this fails (is it ready for traffic?)
        tcpSocket:
          port: 80
        initialDelaySeconds: 5
        periodSeconds: 10
      startupProbe:                # gates liveness/readiness until slow-starting app is up
        exec:
          command: ["cat", "/tmp/started"]
        failureThreshold: 30
        periodSeconds: 5
      lifecycle:                   # hooks run on container lifecycle events
        postStart:                 # runs right after the container starts
          exec:
            command: ["sh", "-c", "echo started"]
        preStop:                   # runs right before the container is terminated (e.g. drain connections)
          exec:
            command: ["sh", "-c", "sleep 5"]
      securityContext:             # per-container security/privilege settings
        runAsUser: 1000
        runAsNonRoot: true
        allowPrivilegeEscalation: false
        readOnlyRootFilesystem: true
        capabilities:
          drop: ["ALL"]            # drop all Linux capabilities (least privilege)
  volumes:                         # storage sources available to be mounted by containers above
    - name: config-vol
      configMap:                   # exposes a ConfigMap's data as files
        name: my-config
    - name: secret-vol
      secret:                      # exposes a Secret's data as files
        secretName: my-secret
    - name: data-vol
      persistentVolumeClaim:       # attaches a persistent disk backed by a PVC
        claimName: my-pvc
    - name: empty-vol
      emptyDir: {}                 # scratch space, created empty, deleted when pod is removed
    - name: host-vol
      hostPath:                    # mounts a path from the underlying node's filesystem
        path: /data
        type: DirectoryOrCreate
  imagePullSecrets:                # credentials for pulling images from private registries
    - name: regcred
```

---

## 2. Deployment

### Simple

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-deployment
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
        - name: app
          image: nginx:latest
```

### Full

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-deployment
  namespace: default
  labels:
    app: my-app
spec:
  replicas: 3                      # desired number of identical pod copies
  revisionHistoryLimit: 10         # how many old ReplicaSets to keep around for rollback
  progressDeadlineSeconds: 600     # marks the rollout as failed if no progress within this time
  minReadySeconds: 5               # pod must stay Ready this long before it's considered available
  paused: false                    # true = stop reconciling changes (useful mid-edit before rollout)
  strategy:                        # how old pods get replaced with new ones on update
    type: RollingUpdate           # RollingUpdate (gradual) | Recreate (kill all, then start all)
    rollingUpdate:
      maxSurge: 25%                # how many extra pods can exist above `replicas` during rollout
      maxUnavailable: 25%          # how many pods can be unavailable during rollout
  selector:                        # which pods this Deployment manages (must match template labels)
    matchLabels:
      app: my-app
  template:                        # the pod spec that gets stamped out `replicas` times
    metadata:
      labels:
        app: my-app
    spec:
      containers:
        - name: app
          image: nginx:1.27
          ports:
            - containerPort: 80
          resources:
            requests: { cpu: "100m", memory: "128Mi" }
            limits: { cpu: "500m", memory: "256Mi" }
          # ... same container fields as Pod above
```

---

## 3. ReplicaSet (rarely created directly — managed by Deployment)

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: my-replicaset
spec:
  replicas: 3                      # desired number of pod copies to keep running at all times
  selector:                        # which pods this ReplicaSet is responsible for counting/replacing
    matchLabels:
      app: my-app
  template:                        # pod spec used to create new replicas when short of `replicas`
    metadata:
      labels:
        app: my-app
    spec:
      containers:
        - name: app
          image: nginx:latest
```

---

## 4. StatefulSet

### Simple

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: my-statefulset
spec:
  serviceName: my-headless-svc
  replicas: 3
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
        - name: app
          image: nginx:latest
```

### Full

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: my-statefulset
  namespace: default
spec:
  serviceName: my-headless-svc     # headless Service that gives each pod a stable DNS name
  replicas: 3
  podManagementPolicy: OrderedReady  # pod startup/scaling order — OrderedReady (one at a time) | Parallel
  updateStrategy:                  # how pods get updated when the template changes
    type: RollingUpdate            # RollingUpdate | OnDelete (only replace pod when manually deleted)
    rollingUpdate:
      partition: 0                 # pods with ordinal >= this number get updated; below it, untouched
  revisionHistoryLimit: 10
  minReadySeconds: 0
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
        - name: app
          image: postgres:16
          ports:
            - containerPort: 5432
          volumeMounts:
            - name: data
              mountPath: /var/lib/postgresql/data
  volumeClaimTemplates:            # a unique PVC is auto-created per pod (data-0, data-1, ...), stable per pod identity
    - metadata:
        name: data
      spec:
        accessModes: ["ReadWriteOnce"]
        storageClassName: standard
        resources:
          requests:
            storage: 10Gi
```

---

## 5. DaemonSet

### Simple

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: my-daemonset
spec:
  selector:
    matchLabels:
      app: my-agent
  template:
    metadata:
      labels:
        app: my-agent
    spec:
      containers:
        - name: agent
          image: fluent/fluentd:latest
```

### Full

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: my-daemonset
  namespace: kube-system
spec:
  selector:                        # runs exactly one matching pod per eligible node
    matchLabels:
      app: my-agent
  updateStrategy:
    type: RollingUpdate            # RollingUpdate | OnDelete
    rollingUpdate:
      maxUnavailable: 1            # how many node-pods can be down at once during rollout
  minReadySeconds: 0
  revisionHistoryLimit: 10
  template:
    metadata:
      labels:
        app: my-agent
    spec:
      tolerations:                 # needed to also schedule onto tainted nodes, e.g. control-plane
        - key: node-role.kubernetes.io/control-plane
          effect: NoSchedule
      containers:
        - name: agent
          image: fluent/fluentd:latest
          resources:
            requests: { cpu: "50m", memory: "64Mi" }
          volumeMounts:
            - name: varlog
              mountPath: /var/log
      volumes:
        - name: varlog
          hostPath:                # gives the agent access to the node's own log files
            path: /var/log
```

---

## 6. Job

### Simple

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: my-job
spec:
  template:
    spec:
      containers:
        - name: worker
          image: busybox:latest
          command: ["echo", "hello"]
      restartPolicy: Never
```

### Full

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: my-job
  namespace: default
spec:
  completions: 5                   # total number of successful pod completions needed
  parallelism: 2                   # how many pods can run at once working toward `completions`
  backoffLimit: 4                  # number of retries before marking the Job failed
  activeDeadlineSeconds: 600       # kills the whole Job if it runs longer than this
  ttlSecondsAfterFinished: 3600    # auto-delete the Job (and its pods) this long after it finishes
  completionMode: NonIndexed        # NonIndexed (any pod success counts) | Indexed (each pod gets a unique index)
  suspend: false                   # true = pause without deleting; pods aren't created until unsuspended
  template:
    metadata:
      labels:
        job-name: my-job
    spec:
      restartPolicy: Never          # what happens to a failed container — Never | OnFailure
      containers:
        - name: worker
          image: busybox:latest
          command: ["sh", "-c", "echo doing work; sleep 5"]
          resources:
            requests: { cpu: "100m", memory: "64Mi" }
```

---

## 7. CronJob

### Simple

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: my-cronjob
spec:
  schedule: "*/5 * * * *"
  jobTemplate:
    spec:
      template:
        spec:
          containers:
            - name: job
              image: busybox:latest
              command: ["echo", "hello"]
          restartPolicy: OnFailure
```

### Full

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: my-cronjob
  namespace: default
spec:
  schedule: "0 2 * * *"             # standard cron syntax for when to run
  timeZone: "UTC"                   # timezone the schedule is evaluated in
  concurrencyPolicy: Forbid         # what to do if the previous run hasn't finished — Allow | Forbid | Replace
  startingDeadlineSeconds: 100      # how late a missed run is still allowed to start
  successfulJobsHistoryLimit: 3     # how many completed Jobs to keep for inspection
  failedJobsHistoryLimit: 1         # how many failed Jobs to keep for inspection
  suspend: false                    # true = stop scheduling new runs without deleting the CronJob
  jobTemplate:                      # the Job spec that gets created on each scheduled trigger
    spec:
      backoffLimit: 3
      template:
        spec:
          containers:
            - name: job
              image: busybox:latest
              command: ["sh", "-c", "echo scheduled task"]
          restartPolicy: OnFailure
```

---

## 8. Service

### Simple (ClusterIP)

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  selector:
    app: my-app
  ports:
    - port: 80
      targetPort: 8080
```

### Full (all types shown separately)

```yaml
# ClusterIP (default) — internal-only virtual IP for load-balancing across matching pods
apiVersion: v1
kind: Service
metadata:
  name: my-service
  namespace: default
  labels:
    app: my-app
  annotations: {}
spec:
  type: ClusterIP                   # how the Service is exposed — ClusterIP | NodePort | LoadBalancer | ExternalName
  selector:                         # which pods receive traffic (matched by labels)
    app: my-app
  ports:
    - name: http
      protocol: TCP                 # TCP | UDP | SCTP
      port: 80                      # port the Service itself listens on
      targetPort: 8080              # port on the pod/container traffic is forwarded to
    - name: https
      protocol: TCP
      port: 443
      targetPort: 8443
  sessionAffinity: None             # stick a client to the same pod — None | ClientIP
  clusterIP: None                   # set literal "None" to make this a headless Service (no load-balancing, direct pod DNS)
---
# NodePort — additionally exposes the Service on a static port on every node's IP
apiVersion: v1
kind: Service
metadata:
  name: my-nodeport-service
spec:
  type: NodePort
  selector:
    app: my-app
  ports:
    - port: 80
      targetPort: 8080
      nodePort: 30080              # externally reachable port on every node — must be 30000-32767
---
# LoadBalancer — provisions an external cloud load balancer pointing at the Service
apiVersion: v1
kind: Service
metadata:
  name: my-lb-service
spec:
  type: LoadBalancer
  selector:
    app: my-app
  ports:
    - port: 80
      targetPort: 8080
  loadBalancerSourceRanges:        # restrict which client IPs are allowed to reach the LB
    - 0.0.0.0/0
---
# ExternalName — DNS-level alias to an external hostname, no proxying/selecting involved
apiVersion: v1
kind: Service
metadata:
  name: my-external-service
spec:
  type: ExternalName
  externalName: api.example.com    # the external DNS name this Service resolves to
```

---

## 9. Ingress

### Simple

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-ingress
spec:
  rules:
    - host: example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: my-service
                port:
                  number: 80
```

### Full

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-ingress
  namespace: default
  annotations:                      # controller-specific behavior (varies by ingress controller, e.g. nginx)
    nginx.ingress.kubernetes.io/rewrite-target: /
    cert-manager.io/cluster-issuer: letsencrypt-prod
spec:
  ingressClassName: nginx           # which Ingress controller should handle this resource
  tls:                              # HTTPS termination config
    - hosts:
        - example.com
      secretName: example-tls       # TLS cert/key Secret to use for these hosts
  rules:                            # HTTP routing rules, evaluated per host
    - host: example.com
      http:
        paths:
          - path: /api
            pathType: Prefix       # how `path` is matched — Prefix | Exact | ImplementationSpecific
            backend:
              service:
                name: api-service
                port:
                  number: 80
          - path: /
            pathType: Prefix
            backend:
              service:
                name: web-service
                port:
                  number: 80
  defaultBackend:                   # where to send requests that match no rule above
    service:
      name: default-service
      port:
        number: 80
```

---

## 10. ConfigMap

### Simple

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: my-config
data:
  key1: value1
```

### Full

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: my-config
  namespace: default
immutable: false                    # true = reject any future updates (perf + safety for static config)
data:                                # plain-text key/value config, injectable as env vars or files
  key1: value1
  key2: value2
  app.properties: |                 # a whole config file's contents can live under one key
    setting1=true
    setting2=false
binaryData:                         # for non-UTF8 content, base64-encoded
  binary-key: base64EncodedContentHere==
```

---

## 11. Secret

### Simple

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: my-secret
type: Opaque
stringData:
  password: mypassword
```

### Full (all common types)

```yaml
# Opaque — generic, arbitrary key/value secret data (default type)
apiVersion: v1
kind: Secret
metadata:
  name: my-secret
  namespace: default
type: Opaque                       # Opaque | kubernetes.io/dockerconfigjson | kubernetes.io/tls | kubernetes.io/basic-auth | kubernetes.io/ssh-auth | kubernetes.io/service-account-token
immutable: false
stringData:               # write plaintext here — the API server base64-encodes it for you on save
  username: admin
  password: s3cr3t
data:                      # values here must already be base64-encoded by you
  key: dmFsdWU=
---
# docker registry credentials — used via imagePullSecrets to pull private images
apiVersion: v1
kind: Secret
metadata:
  name: regcred
type: kubernetes.io/dockerconfigjson
data:
  .dockerconfigjson: eyJhdXRocyI6IHt9fQ==
---
# TLS certificate — used by Ingress or other TLS-terminating resources
apiVersion: v1
kind: Secret
metadata:
  name: tls-secret
type: kubernetes.io/tls
data:
  tls.crt: <base64 cert>
  tls.key: <base64 key>
---
# basic-auth — username/password credential pair
apiVersion: v1
kind: Secret
metadata:
  name: basic-auth-secret
type: kubernetes.io/basic-auth
stringData:
  username: admin
  password: t0p-Secret
---
# ssh-auth — private key for SSH-based access (e.g. git clone over SSH)
apiVersion: v1
kind: Secret
metadata:
  name: ssh-key-secret
type: kubernetes.io/ssh-auth
data:
  ssh-privatekey: <base64 private key>
```

---

## 12. PersistentVolume

### Simple

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: my-pv
spec:
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: /data
```

### Full

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: my-pv
  labels:
    type: local
spec:
  capacity:
    storage: 10Gi                   # total size of this volume
  volumeMode: Filesystem            # how it's presented to the pod — Filesystem | Block
  accessModes:                      # how many nodes/pods can mount it, and how
    - ReadWriteOnce                 # ReadWriteOnce | ReadOnlyMany | ReadWriteMany | ReadWriteOncePod
  persistentVolumeReclaimPolicy: Retain  # what happens to the underlying storage when its PVC is deleted — Retain | Delete | Recycle
  storageClassName: manual          # must match the PVC's storageClassName to bind
  mountOptions:                     # extra mount flags passed to the underlying filesystem
    - hard
    - nfsvers=4.1
  nodeAffinity:                     # restricts which nodes can access this volume (for local/topology-bound storage)
    required:
      nodeSelectorTerms:
        - matchExpressions:
            - key: kubernetes.io/hostname
              operator: In
              values: ["node1"]
  # Choose ONE volume source (where the data actually lives):
  hostPath:
    path: /data
    type: DirectoryOrCreate
  # nfs:
  #   server: nfs.example.com
  #   path: /exports/data
  # csi:
  #   driver: ebs.csi.aws.com
  #   volumeHandle: vol-0123456789
  #   fsType: ext4
```

---

## 13. PersistentVolumeClaim

### Simple

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
```

### Full

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-pvc
  namespace: default
spec:
  accessModes:                     # ReadWriteOnce | ReadOnlyMany | ReadWriteMany | ReadWriteOncePod
    - ReadWriteOnce
  volumeMode: Filesystem            # Filesystem | Block
  storageClassName: standard        # which StorageClass provisions/matches the backing volume
  resources:
    requests:
      storage: 5Gi                  # minimum size requested
    limits:
      storage: 20Gi                 # optional cap (rarely used on PVCs)
  selector:                         # manually bind to a PV matching these labels (bypasses dynamic provisioning)
    matchLabels:
      type: local
  dataSource:                      # clone from snapshot or another PVC
    name: my-snapshot
    kind: VolumeSnapshot
    apiGroup: snapshot.storage.k8s.io
```

---

## 14. StorageClass

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-ssd
provisioner: ebs.csi.aws.com        # driver that dynamically creates volumes for this class — e.g. kubernetes.io/aws-ebs, pd.csi.storage.gke.io
parameters:                         # driver-specific tuning
  type: gp3
  fsType: ext4
reclaimPolicy: Delete               # what happens to the disk when its PVC is deleted — Retain | Delete
volumeBindingMode: WaitForFirstConsumer  # when to actually provision — Immediate | WaitForFirstConsumer (waits until a pod needs it, avoids AZ mismatch)
allowVolumeExpansion: true          # lets PVCs using this class be resized later
mountOptions:
  - debug
```

---

## 15. Namespace

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: my-namespace
  labels:                          # used for namespace-level selectors (e.g. NetworkPolicy namespaceSelector)
    env: production
  annotations:
    description: "production workloads"
```

---

## 16. ServiceAccount

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: my-sa
  namespace: default
automountServiceAccountToken: true  # whether pods using this SA auto-get an API token mounted
imagePullSecrets:                   # registry credentials auto-attached to pods using this SA
  - name: regcred
secrets:                            # legacy: pre-created token Secrets associated with this SA
  - name: my-sa-token
```

---

## 17. Role / RoleBinding (namespaced RBAC)

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
  namespace: default                # Role permissions only apply within this namespace
rules:
  - apiGroups: [""]                # "" = core API group (pods, services, configmaps, etc.)
    resources: ["pods", "pods/log"]
    verbs: ["get", "list", "watch"]  # allowed API actions
  - apiGroups: ["apps"]
    resources: ["deployments"]
    verbs: ["get", "list"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods
  namespace: default
subjects:                          # who is granted the permissions
  - kind: ServiceAccount
    name: my-sa
    namespace: default
  - kind: User
    name: jane@example.com
    apiGroup: rbac.authorization.k8s.io
  - kind: Group
    name: developers
    apiGroup: rbac.authorization.k8s.io
roleRef:                           # the Role being granted to the subjects above
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

---

## 18. ClusterRole / ClusterRoleBinding (cluster-wide RBAC)

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: node-reader                 # unlike Role, applies cluster-wide (or reusable across namespaces via RoleBinding)
rules:
  - apiGroups: [""]
    resources: ["nodes"]            # nodes are cluster-scoped, so this needs a ClusterRole not a Role
    verbs: ["get", "list", "watch"]
  - nonResourceURLs: ["/healthz"]   # can also grant access to non-resource API endpoints
    verbs: ["get"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: read-nodes-global
subjects:
  - kind: ServiceAccount
    name: my-sa
    namespace: default
roleRef:
  kind: ClusterRole
  name: node-reader
  apiGroup: rbac.authorization.k8s.io
```

---

## 19. NetworkPolicy

### Simple (deny all ingress)

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all-ingress
spec:
  podSelector: {}
  policyTypes:
    - Ingress
```

### Full

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: my-network-policy
  namespace: default
spec:
  podSelector:                     # which pods this policy applies to (empty {} = all pods in namespace)
    matchLabels:
      app: my-app
  policyTypes:                     # which traffic direction(s) this policy governs — subset of: Ingress, Egress
    - Ingress
    - Egress
  ingress:                          # allowed inbound traffic (if omitted entirely, all ingress is blocked)
    - from:
        - podSelector:              # allow from pods with this label, in the same namespace
            matchLabels:
              role: frontend
        - namespaceSelector:        # allow from any pod in namespaces with this label
            matchLabels:
              env: production
        - ipBlock:                  # allow from this CIDR range
            cidr: 10.0.0.0/24
            except:
              - 10.0.0.1/32
      ports:
        - protocol: TCP
          port: 80
  egress:                           # allowed outbound traffic (if omitted entirely, all egress is blocked)
    - to:
        - podSelector:
            matchLabels:
              role: database
      ports:
        - protocol: TCP
          port: 5432
```

---

## 20. HorizontalPodAutoscaler

### Simple

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: my-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: my-deployment
  minReplicas: 2
  maxReplicas: 10
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
```

### Full

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: my-hpa
  namespace: default
spec:
  scaleTargetRef:                  # the workload this HPA scales
    apiVersion: apps/v1
    kind: Deployment
    name: my-deployment
  minReplicas: 2                   # floor on replica count
  maxReplicas: 10                  # ceiling on replica count
  metrics:                         # signals used to decide when to scale, can combine several
    - type: Resource               # built-in cpu/memory usage — Resource | Pods | Object | External | ContainerResource
      resource:
        name: cpu
        target:
          type: Utilization         # target expressed as % of requested resource — Utilization | AverageValue | Value
          averageUtilization: 70
    - type: Resource
      resource:
        name: memory
        target:
          type: AverageValue        # target expressed as an absolute average value per pod
          averageValue: 500Mi
    - type: Pods                    # custom metric averaged across all target pods
      pods:
        metric:
          name: packets-per-second
        target:
          type: AverageValue
          averageValue: 1k
    - type: External                # metric from outside the cluster (e.g. a queue depth)
      external:
        metric:
          name: queue_messages_ready
          selector:
            matchLabels:
              queue: worker-tasks
        target:
          type: AverageValue
          averageValue: 30
  behavior:                        # fine-tune scale-up/down speed to avoid thrashing
    scaleDown:
      stabilizationWindowSeconds: 300  # wait this long of sustained low usage before scaling down
      policies:
        - type: Percent
          value: 50                 # scale down by at most 50% of current replicas per period
          periodSeconds: 60
    scaleUp:
      stabilizationWindowSeconds: 0
      policies:
        - type: Percent
          value: 100                # can double replica count per period
          periodSeconds: 15
```

---

## 21. VerticalPodAutoscaler (requires VPA CRD/controller installed)

```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: my-vpa
spec:
  targetRef:                       # the workload whose container resource requests/limits get adjusted
    apiVersion: apps/v1
    kind: Deployment
    name: my-deployment
  updatePolicy:
    updateMode: "Auto"             # how recommendations get applied — Off (recommend only) | Initial | Recreate | Auto
  resourcePolicy:
    containerPolicies:
      - containerName: app
        minAllowed:                # floor on what VPA can set
          cpu: 100m
          memory: 128Mi
        maxAllowed:                # ceiling on what VPA can set
          cpu: 1
          memory: 1Gi
```

---

## 22. PodDisruptionBudget

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: my-pdb
spec:
  minAvailable: 2                  # minimum pods that must stay up during voluntary disruptions (or use maxUnavailable instead)
  # maxUnavailable: 1
  selector:                        # which pods this budget protects
    matchLabels:
      app: my-app
```

---

## 23. ResourceQuota

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: my-quota
  namespace: default               # quotas apply per-namespace
spec:
  hard:                            # hard caps on total resource usage/object counts in the namespace
    requests.cpu: "10"
    requests.memory: 20Gi
    limits.cpu: "20"
    limits.memory: 40Gi
    pods: "50"
    persistentvolumeclaims: "10"
    services.loadbalancers: "2"
    count/deployments.apps: "20"
  scopeSelector:                   # restrict the quota to only objects matching this scope
    matchExpressions:
      - operator: In
        scopeName: PriorityClass
        values: ["high"]
```

---

## 24. LimitRange

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: my-limits
  namespace: default
spec:
  limits:
    - type: Container               # applies to individual containers
      default:                      # limit auto-applied if a container doesn't specify one
        cpu: "500m"
        memory: "256Mi"
      defaultRequest:                # request auto-applied if a container doesn't specify one
        cpu: "100m"
        memory: "128Mi"
      max:                           # highest a container may request/limit
        cpu: "2"
        memory: "1Gi"
      min:                           # lowest a container may request/limit
        cpu: "50m"
        memory: "64Mi"
    - type: Pod                      # aggregate caps across all containers in a pod
      max:
        cpu: "4"
        memory: "2Gi"
    - type: PersistentVolumeClaim    # caps on PVC sizes in this namespace
      max:
        storage: 50Gi
      min:
        storage: 1Gi
```

---

## 25. Endpoints / EndpointSlice (manual, for services without selectors)

```yaml
apiVersion: v1
kind: Endpoints
metadata:
  name: my-service          # must match Service name — manually wires a Service to external/static IPs
subsets:
  - addresses:
      - ip: 192.168.1.1
    ports:
      - port: 9376
---
apiVersion: discovery.k8s.io/v1
kind: EndpointSlice          # modern, scalable replacement for Endpoints
metadata:
  name: my-service-ab12
  labels:
    kubernetes.io/service-name: my-service   # links this slice back to its Service
addressType: IPv4
ports:
  - name: http
    protocol: TCP
    port: 9376
endpoints:
  - addresses: ["192.168.1.1"]
```

---

## 26. PriorityClass

```yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: high-priority
value: 1000000                     # higher number = higher scheduling priority
globalDefault: false               # true = applied to any pod that doesn't specify a priorityClassName
preemptionPolicy: PreemptLowerPriority  # can this priority preempt lower-priority pods to get scheduled? — PreemptLowerPriority | Never
description: "Used for critical production workloads"
```

---

## 27. CustomResourceDefinition (CRD)

```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: widgets.example.com         # must be <plural>.<group>
spec:
  group: example.com                # API group the new resource lives under
  names:
    kind: Widget
    listKind: WidgetList
    plural: widgets
    singular: widget
    shortNames: ["wg"]              # kubectl get wg
  scope: Namespaced               # Namespaced | Cluster
  versions:
    - name: v1
      served: true                 # true | false — whether this version is enabled via the API
      storage: true                # true | false — exactly one version must be the storage version
      schema:                      # OpenAPI validation schema for instances of this resource
        openAPIV3Schema:
          type: object
          properties:
            spec:
              type: object
              properties:
                size:
                  type: string
                replicas:
                  type: integer
      subresources:
        status: {}                 # enables a separate /status subresource (common controller pattern)
      additionalPrinterColumns:    # extra columns shown by `kubectl get`
        - name: Size
          type: string
          jsonPath: .spec.size
```

---

## 28. Pod Security Admission label (Namespace-level, replaces PodSecurityPolicy)

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: restricted-ns
  labels:                                              # built-in admission control enforcing Pod Security Standards
    pod-security.kubernetes.io/enforce: restricted      # blocks non-conforming pods — privileged | baseline | restricted
    pod-security.kubernetes.io/enforce-version: latest
    pod-security.kubernetes.io/warn: restricted         # warns (doesn't block) on non-conformance
    pod-security.kubernetes.io/audit: restricted         # logs to audit trail on non-conformance
```

---

## 29. MutatingWebhookConfiguration / ValidatingWebhookConfiguration

```yaml
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingWebhookConfiguration
metadata:
  name: my-validating-webhook
webhooks:
  - name: validate.example.com
    clientConfig:                   # how the API server reaches your webhook service
      service:
        name: webhook-service
        namespace: default
        path: "/validate"
      caBundle: <base64 CA cert>    # CA cert used to trust the webhook's TLS endpoint
    rules:                          # which API requests trigger this webhook
      - apiGroups: [""]
        apiVersions: ["v1"]
        operations: ["CREATE", "UPDATE"]   # CREATE | UPDATE | DELETE | CONNECT | *
        resources: ["pods"]
        scope: "Namespaced"         # Namespaced | Cluster | *
    admissionReviewVersions: ["v1"]
    sideEffects: None               # does the webhook have side effects outside the admission request? — None | NoneOnDryRun | Some | Unknown
    failurePolicy: Fail            # what happens if the webhook is unreachable — Fail (block request) | Ignore (allow request)
    timeoutSeconds: 5
```

---

## Quick reference: common `kubectl` generators

```bash
kubectl create deployment my-app --image=nginx --replicas=3 --dry-run=client -o yaml
kubectl create service clusterip my-svc --tcp=80:8080 --dry-run=client -o yaml
kubectl create configmap my-config --from-literal=key=value --dry-run=client -o yaml
kubectl create secret generic my-secret --from-literal=password=pass --dry-run=client -o yaml
kubectl create job my-job --image=busybox -- echo hello --dry-run=client -o yaml
kubectl create cronjob my-cj --image=busybox --schedule="*/5 * * * *" -- echo hi --dry-run=client -o yaml
kubectl create role my-role --verb=get,list --resource=pods --dry-run=client -o yaml
kubectl create rolebinding my-rb --role=my-role --serviceaccount=default:my-sa --dry-run=client -o yaml
kubectl create ingress my-ing --rule="example.com/*=my-svc:80" --dry-run=client -o yaml
```

---

Ready for your first task — send it over.