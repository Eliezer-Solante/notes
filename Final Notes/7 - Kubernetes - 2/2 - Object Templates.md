# Kubernetes Object Templates Reference

For each object: a **Simple** minimal working template, and a **Full** template showing (nearly) all commonly-used fields, with inline comments. `apiVersion` values reflect current stable Kubernetes (1.29+).

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
  namespace: default
  labels:
    app: my-app
    tier: frontend
  annotations:
    description: "example pod"
spec:
  restartPolicy: Always            # Always | OnFailure | Never
  terminationGracePeriodSeconds: 30
  serviceAccountName: my-sa
  nodeSelector:
    disktype: ssd
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
          - matchExpressions:
              - key: kubernetes.io/os
                operator: In
                values: ["linux"]
    podAntiAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
        - weight: 100
          podAffinityTerm:
            labelSelector:
              matchLabels:
                app: my-app
            topologyKey: kubernetes.io/hostname
  tolerations:
    - key: "key1"
      operator: "Equal"
      value: "value1"
      effect: "NoSchedule"
  topologySpreadConstraints:
    - maxSkew: 1
      topologyKey: topology.kubernetes.io/zone
      whenUnsatisfiable: DoNotSchedule
      labelSelector:
        matchLabels:
          app: my-app
  initContainers:
    - name: init-setup
      image: busybox:latest
      command: ["sh", "-c", "echo init"]
  containers:
    - name: app
      image: nginx:1.27
      imagePullPolicy: IfNotPresent   # Always | IfNotPresent | Never
      command: ["/bin/sh"]
      args: ["-c", "nginx -g 'daemon off;'"]
      workingDir: /app
      ports:
        - name: http
          containerPort: 80
          protocol: TCP
      env:
        - name: ENV_VAR
          value: "value1"
        - name: FROM_SECRET
          valueFrom:
            secretKeyRef:
              name: my-secret
              key: password
        - name: FROM_CONFIGMAP
          valueFrom:
            configMapKeyRef:
              name: my-config
              key: setting
        - name: POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
      envFrom:
        - configMapRef:
            name: my-config
        - secretRef:
            name: my-secret
      resources:
        requests:
          cpu: "250m"
          memory: "128Mi"
        limits:
          cpu: "500m"
          memory: "256Mi"
      volumeMounts:
        - name: config-vol
          mountPath: /etc/config
        - name: data-vol
          mountPath: /data
      livenessProbe:
        httpGet:
          path: /healthz
          port: 80
        initialDelaySeconds: 10
        periodSeconds: 10
        timeoutSeconds: 1
        failureThreshold: 3
      readinessProbe:
        tcpSocket:
          port: 80
        initialDelaySeconds: 5
        periodSeconds: 10
      startupProbe:
        exec:
          command: ["cat", "/tmp/started"]
        failureThreshold: 30
        periodSeconds: 5
      lifecycle:
        postStart:
          exec:
            command: ["sh", "-c", "echo started"]
        preStop:
          exec:
            command: ["sh", "-c", "sleep 5"]
      securityContext:
        runAsUser: 1000
        runAsNonRoot: true
        allowPrivilegeEscalation: false
        readOnlyRootFilesystem: true
        capabilities:
          drop: ["ALL"]
  volumes:
    - name: config-vol
      configMap:
        name: my-config
    - name: secret-vol
      secret:
        secretName: my-secret
    - name: data-vol
      persistentVolumeClaim:
        claimName: my-pvc
    - name: empty-vol
      emptyDir: {}
    - name: host-vol
      hostPath:
        path: /data
        type: DirectoryOrCreate
  imagePullSecrets:
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
  replicas: 3
  revisionHistoryLimit: 10
  progressDeadlineSeconds: 600
  minReadySeconds: 5
  paused: false
  strategy:
    type: RollingUpdate           # RollingUpdate | Recreate
    rollingUpdate:
      maxSurge: 25%
      maxUnavailable: 25%
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
  serviceName: my-headless-svc     # must match a headless Service
  replicas: 3
  podManagementPolicy: OrderedReady  # OrderedReady | Parallel
  updateStrategy:
    type: RollingUpdate            # RollingUpdate | OnDelete
    rollingUpdate:
      partition: 0
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
  volumeClaimTemplates:
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
  selector:
    matchLabels:
      app: my-agent
  updateStrategy:
    type: RollingUpdate            # RollingUpdate | OnDelete
    rollingUpdate:
      maxUnavailable: 1
  minReadySeconds: 0
  revisionHistoryLimit: 10
  template:
    metadata:
      labels:
        app: my-agent
    spec:
      tolerations:
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
          hostPath:
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
  completions: 5
  parallelism: 2
  backoffLimit: 4
  activeDeadlineSeconds: 600
  ttlSecondsAfterFinished: 3600
  completionMode: NonIndexed        # NonIndexed | Indexed
  suspend: false
  template:
    metadata:
      labels:
        job-name: my-job
    spec:
      restartPolicy: Never          # Never | OnFailure
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
  schedule: "0 2 * * *"             # cron syntax
  timeZone: "UTC"
  concurrencyPolicy: Forbid         # Allow | Forbid | Replace
  startingDeadlineSeconds: 100
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 1
  suspend: false
  jobTemplate:
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
# ClusterIP (default)
apiVersion: v1
kind: Service
metadata:
  name: my-service
  namespace: default
  labels:
    app: my-app
  annotations: {}
spec:
  type: ClusterIP                   # ClusterIP | NodePort | LoadBalancer | ExternalName
  selector:
    app: my-app
  ports:
    - name: http
      protocol: TCP
      port: 80
      targetPort: 8080
    - name: https
      protocol: TCP
      port: 443
      targetPort: 8443
  sessionAffinity: None             # None | ClientIP
  clusterIP: None                   # set "None" for headless service
---
# NodePort
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
      nodePort: 30080              # 30000-32767
---
# LoadBalancer
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
  loadBalancerSourceRanges:
    - 0.0.0.0/0
---
# ExternalName
apiVersion: v1
kind: Service
metadata:
  name: my-external-service
spec:
  type: ExternalName
  externalName: api.example.com
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
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
    cert-manager.io/cluster-issuer: letsencrypt-prod
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - example.com
      secretName: example-tls
  rules:
    - host: example.com
      http:
        paths:
          - path: /api
            pathType: Prefix       # Prefix | Exact | ImplementationSpecific
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
  defaultBackend:
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
immutable: false
data:
  key1: value1
  key2: value2
  app.properties: |
    setting1=true
    setting2=false
binaryData:
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
# Opaque (generic)
apiVersion: v1
kind: Secret
metadata:
  name: my-secret
  namespace: default
type: Opaque                       # Opaque | kubernetes.io/dockerconfigjson | kubernetes.io/tls | kubernetes.io/basic-auth | kubernetes.io/ssh-auth | kubernetes.io/service-account-token
immutable: false
stringData:               # plaintext, auto-encoded by kube-apiserver
  username: admin
  password: s3cr3t
data:                      # already base64-encoded values
  key: dmFsdWU=
---
# docker registry credentials
apiVersion: v1
kind: Secret
metadata:
  name: regcred
type: kubernetes.io/dockerconfigjson
data:
  .dockerconfigjson: eyJhdXRocyI6IHt9fQ==
---
# TLS certificate
apiVersion: v1
kind: Secret
metadata:
  name: tls-secret
type: kubernetes.io/tls
data:
  tls.crt: <base64 cert>
  tls.key: <base64 key>
---
# basic-auth
apiVersion: v1
kind: Secret
metadata:
  name: basic-auth-secret
type: kubernetes.io/basic-auth
stringData:
  username: admin
  password: t0p-Secret
---
# ssh-auth
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
    storage: 10Gi
  volumeMode: Filesystem            # Filesystem | Block
  accessModes:
    - ReadWriteOnce                 # ReadWriteOnce | ReadOnlyMany | ReadWriteMany | ReadWriteOncePod
  persistentVolumeReclaimPolicy: Retain  # Retain | Delete | Recycle
  storageClassName: manual
  mountOptions:
    - hard
    - nfsvers=4.1
  nodeAffinity:
    required:
      nodeSelectorTerms:
        - matchExpressions:
            - key: kubernetes.io/hostname
              operator: In
              values: ["node1"]
  # Choose ONE volume source:
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
  storageClassName: standard
  resources:
    requests:
      storage: 5Gi
    limits:
      storage: 20Gi
  selector:
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
provisioner: ebs.csi.aws.com        # e.g. kubernetes.io/aws-ebs, pd.csi.storage.gke.io
parameters:
  type: gp3
  fsType: ext4
reclaimPolicy: Delete               # Retain | Delete
volumeBindingMode: WaitForFirstConsumer  # Immediate | WaitForFirstConsumer
allowVolumeExpansion: true
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
  labels:
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
automountServiceAccountToken: true
imagePullSecrets:
  - name: regcred
secrets:
  - name: my-sa-token
```

---

## 17. Role / RoleBinding (namespaced RBAC)

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
  namespace: default
rules:
  - apiGroups: [""]                # "" = core API group
    resources: ["pods", "pods/log"]
    verbs: ["get", "list", "watch"]
  - apiGroups: ["apps"]
    resources: ["deployments"]
    verbs: ["get", "list"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods
  namespace: default
subjects:
  - kind: ServiceAccount
    name: my-sa
    namespace: default
  - kind: User
    name: jane@example.com
    apiGroup: rbac.authorization.k8s.io
  - kind: Group
    name: developers
    apiGroup: rbac.authorization.k8s.io
roleRef:
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
  name: node-reader
rules:
  - apiGroups: [""]
    resources: ["nodes"]
    verbs: ["get", "list", "watch"]
  - nonResourceURLs: ["/healthz"]
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
  podSelector:
    matchLabels:
      app: my-app
  policyTypes:                     # subset of: Ingress, Egress
    - Ingress
    - Egress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              role: frontend
        - namespaceSelector:
            matchLabels:
              env: production
        - ipBlock:
            cidr: 10.0.0.0/24
            except:
              - 10.0.0.1/32
      ports:
        - protocol: TCP
          port: 80
  egress:
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
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: my-deployment
  minReplicas: 2
  maxReplicas: 10
  metrics:
    - type: Resource               # Resource | Pods | Object | External | ContainerResource
      resource:
        name: cpu
        target:
          type: Utilization         # Utilization | AverageValue | Value
          averageUtilization: 70
    - type: Resource
      resource:
        name: memory
        target:
          type: AverageValue        # Utilization | AverageValue | Value
          averageValue: 500Mi
    - type: Pods
      pods:
        metric:
          name: packets-per-second
        target:
          type: AverageValue        # Pods metrics only support AverageValue
          averageValue: 1k
    - type: External
      external:
        metric:
          name: queue_messages_ready
          selector:
            matchLabels:
              queue: worker-tasks
        target:
          type: AverageValue        # AverageValue | Value
          averageValue: 30
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
        - type: Percent
          value: 50
          periodSeconds: 60
    scaleUp:
      stabilizationWindowSeconds: 0
      policies:
        - type: Percent
          value: 100
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
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: my-deployment
  updatePolicy:
    updateMode: "Auto"             # Off | Initial | Recreate | Auto
  resourcePolicy:
    containerPolicies:
      - containerName: app
        minAllowed:
          cpu: 100m
          memory: 128Mi
        maxAllowed:
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
  minAvailable: 2                  # or use maxUnavailable instead
  # maxUnavailable: 1
  selector:
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
  namespace: default
spec:
  hard:
    requests.cpu: "10"
    requests.memory: 20Gi
    limits.cpu: "20"
    limits.memory: 40Gi
    pods: "50"
    persistentvolumeclaims: "10"
    services.loadbalancers: "2"
    count/deployments.apps: "20"
  scopeSelector:
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
    - type: Container
      default:
        cpu: "500m"
        memory: "256Mi"
      defaultRequest:
        cpu: "100m"
        memory: "128Mi"
      max:
        cpu: "2"
        memory: "1Gi"
      min:
        cpu: "50m"
        memory: "64Mi"
    - type: Pod
      max:
        cpu: "4"
        memory: "2Gi"
    - type: PersistentVolumeClaim
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
  name: my-service          # must match Service name
subsets:
  - addresses:
      - ip: 192.168.1.1
    ports:
      - port: 9376
---
apiVersion: discovery.k8s.io/v1
kind: EndpointSlice
metadata:
  name: my-service-ab12
  labels:
    kubernetes.io/service-name: my-service
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
value: 1000000
globalDefault: false
preemptionPolicy: PreemptLowerPriority  # PreemptLowerPriority | Never
description: "Used for critical production workloads"
```

---

## 27. CustomResourceDefinition (CRD)

```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: widgets.example.com
spec:
  group: example.com
  names:
    kind: Widget
    listKind: WidgetList
    plural: widgets
    singular: widget
    shortNames: ["wg"]
  scope: Namespaced               # Namespaced | Cluster
  versions:
    - name: v1
      served: true                 # true | false — whether this version is enabled via the API
      storage: true                # true | false — exactly one version must be the storage version
      schema:
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
        status: {}
      additionalPrinterColumns:
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
  labels:
    pod-security.kubernetes.io/enforce: restricted   # privileged | baseline | restricted
    pod-security.kubernetes.io/enforce-version: latest
    pod-security.kubernetes.io/warn: restricted
    pod-security.kubernetes.io/audit: restricted
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
    clientConfig:
      service:
        name: webhook-service
        namespace: default
        path: "/validate"
      caBundle: <base64 CA cert>
    rules:
      - apiGroups: [""]
        apiVersions: ["v1"]
        operations: ["CREATE", "UPDATE"]   # CREATE | UPDATE | DELETE | CONNECT | *
        resources: ["pods"]
        scope: "Namespaced"         # Namespaced | Cluster | *
    admissionReviewVersions: ["v1"]
    sideEffects: None               # None | NoneOnDryRun | Some | Unknown
    failurePolicy: Fail            # Fail | Ignore
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