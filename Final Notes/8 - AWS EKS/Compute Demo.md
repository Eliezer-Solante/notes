Here's the full walkthrough with the actual commands and manifests included.

## Part 1 — Managed Node Group

**Check existing nodes**

```sh
k get no
```

```
NAME                                          STATUS   ROLES    AGE    VERSION
i-05b0938045882bc66.us-west-2.compute.internal   Ready   <none>   4h34m   v1.29.0-eks-5e0fdde
i-0b67dcfad12062f1d.us-west-2.compute.internal   Ready   <none>   4h34m   v1.29.0-eks-5e0fdde
```

**Check the Node Group's config**

```sh
eksdemo get mng -c kodekloud
```

```
| Age     | Status | Name | Nodes | Min | Max | Version                | Type       | Instance(s) |
| 4 hours | ACTIVE | main | 2     | 0   | 10  | ami-09167e9f270af0d98   | ON_DEMAND  | t3.large    |
```

**Attempt to raise desired capacity beyond max (fails)**

```sh
eksdemo update mng main -c kodekloud --max 1
```

```
Error: operation error EKS: UpdateNodegroupConfig, ...
InvalidParameterException: desired capacity 2 can't be greater than max size 1
```

**Correct call — raise max in the same command**

```sh
eksdemo update mng main -c kodekloud --max 1 -N 1
```

```
Updating nodegroup with 1 Nodes, 10 Max...done
```

## Part 2 — Deploying to Fargate

**Manifest: `fargate.deploy.yaml`**

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: fargate
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: fargate-nginx
  namespace: fargate
spec:
  replicas: 1
  selector:
    matchLabels:
      app: fargate-nginx
  template:
    metadata:
      labels:
        app: fargate-nginx
    spec:
      terminationGracePeriodSeconds: 0
      containers:
        - name: fargate-nginx
          image: public.ecr.aws/nginx/nginx:latest
```

**Apply it**

```sh
k apply -f fargate.deploy.yaml
```

```
namespace/fargate created
deployment.apps/fargate-nginx created
```

**Check the Pod (initially Pending — Fargate provisioning a micro-VM)**

```sh
k get po -n fargate
```

```
NAME                             READY   STATUS    RESTARTS   AGE
fargate-nginx-58f48f6b79-p46sq   0/1     Pending   0          23s
```

**Inspect the Pod spec — confirms Fargate's own scheduler handled it**

```sh
k get po -n fargate fargate-nginx-58f48f6b79-p46sq -o yaml | less
```

```yaml
schedulerName: fargate-scheduler
tolerations:
  - effect: NoExecute
    key: node.kubernetes.io/not-ready
    operator: Exists
    tolerationSeconds: 300
  - effect: NoExecute
    key: node.kubernetes.io/unreachable
    operator: Exists
```

**Once running, check nodes — a new Fargate node appears**

```sh
k get po -n fargate
```

```
NAME                             READY   STATUS    RESTARTS   AGE
fargate-nginx-58f48f6b79-p46sq   1/1     Running   0          81s
```

```sh
k get no
```

```
NAME                                                STATUS                  ROLES     AGE   VERSION
fargate-ip-192-168-138-113.us-west-2.compute.internal   Ready                 <none>   40s   v1.29.0-eks-680e576
i-05b0938045882bc66.us-west-2.compute.internal          Ready                 <none>   4h38m v1.29.0-eks-5e0fdde
i-0b67dcfad12062f1d.us-west-2.compute.internal          Ready,SchedulingDisabled <none> 4h38m v1.29.0-eks-5e0fdde
```

**Scale to 5 replicas — each gets its own dedicated Fargate node**

```sh
k scale deploy -n fargate fargate-nginx --replicas 5
```

```
deployment.apps/fargate-nginx scaled
```

```sh
k get po -n fargate
```

```
NAME                             READY   STATUS    RESTARTS   AGE
fargate-nginx-58f48f6b79-7ggz5   0/1     Pending   0          4s
fargate-nginx-58f48f6b79-gldtv   0/1     Pending   0          4s
fargate-nginx-58f48f6b79-h8pzw   0/1     Pending   0          4s
fargate-nginx-58f48f6b79-p46sq   1/1     Running   0          2m24s
fargate-nginx-58f48f6b79-sjzx2   0/1     Pending   0          4s
```

## Part 3 — Comparison Deployment on regular EC2 nodes

**Manifest: `nginx.deploy.yaml`**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      terminationGracePeriodSeconds: 0
      containers:
        - name: nginx
          image: public.ecr.aws/nginx/nginx:latest
```

**Apply and check**

```sh
k apply -f nginx.deploy.yaml
```

```
deployment.apps/nginx created
```

```sh
k get po
```

```
NAME                     READY   STATUS              RESTARTS   AGE
nginx-56cd7bd595-8lqdb   0/1     ContainerCreating   0          2s
nginx-56cd7bd595-m5pbs   0/1     ContainerCreating   0          2s
```

**Full-cluster view — Fargate, EC2, and system Pods together**

```sh
k get po -A
```

```
NAMESPACE     NAME                             READY   STATUS    RESTARTS   AGE
default       nginx-56cd7bd595-8lqdb           1/1     Running   0          48s
default       nginx-56cd7bd595-m5pbs           1/1     Running   0          48s
fargate       fargate-nginx-58f48f6b79-7ggz5   1/1     Running   0          91s
fargate       fargate-nginx-58f48f6b79-gldtv   1/1     Running   0          91s
fargate       fargate-nginx-58f48f6b79-h8pzw   1/1     Running   0          91s
fargate       fargate-nginx-58f48f6b79-p46sq   1/1     Running   0          3m51s
fargate       fargate-nginx-58f48f6b79-sjzx2   1/1     Running   0          91s
karpenter     karpenter-cc49c7685-87t2f        1/1     Running   0          9m58s
kube-system   aws-node-vh6tz                   2/2     Running   0          4h40m
kube-system   coredns-5b8cc885bc-smzk8         1/1     Running   0          4m7s
kube-system   coredns-5b8cc885bc-tw6m4         1/1     Running   0          3m6s
kube-system   kube-proxy-lp9np                 1/1     Running   0          4h40m
```

## Part 4 — Karpenter provisions nodes

**Scale nginx to force new node provisioning**

```sh
k scale deploy nginx --replicas 25
```

```
deployment.apps/nginx scaled
```

```sh
k get po -A
```

_(new nginx Pods appear, several `Pending` while Karpenter reacts)_

**Add a CPU resource request so Karpenter can size instances correctly**

```sh
k edit deployments.apps nginx
```

```yaml
spec:
  containers:
    - image: public.ecr.aws/nginx/nginx:latest
      imagePullPolicy: Always
      name: nginx
      resources:
        requests:
          cpu: '0.5'
```

**After rollout, check nodes — mix of Fargate, original EC2, and Karpenter-provisioned nodes**

```sh
k get no
```

```
NAME                                                     STATUS   ROLES    AGE     VERSION
fargate-ip-192-168-109-192.us-west-2.compute.internal    Ready    <none>   5m47s   v1.29.0-eks-680e576
fargate-ip-192-168-138-113.us-west-2.compute.internal    Ready    <none>   8m10s   v1.29.0-eks-680e576
fargate-ip-192-168-141-171.us-west-2.compute.internal    Ready    <none>   5m50s   v1.29.0-eks-680e576
fargate-ip-192-168-182-102.us-west-2.compute.internal    Ready    <none>   5m45s   v1.29.0-eks-680e576
fargate-ip-192-168-99-244.us-west-2.compute.internal     Ready    <none>   5m50s   v1.29.0-eks-680e576
i-048b7c8a501cda49d.us-west-2.compute.internal            Ready    <none>   2m7s    v1.29.0-eks-5e0fdde
i-05b0938045882bc66.us-west-2.compute.internal            Ready    <none>   4h45m   v1.29.0-eks-5e0fdde
i-0c1d8c0722d4758b1.us-west-2.compute.internal            Ready    <none>   86s     v1.29.0-eks-5e0fdde
```

**Inspect a Karpenter-provisioned node — shows it chose the instance type itself**

```sh
k get no i-0c1d8c0722d4758b1.us-west-2.compute.internal -o yaml | less
```

```yaml
finalizers:
  - karpenter.sh/termination
labels:
  beta.kubernetes.io/arch: amd64
  beta.kubernetes.io/instance-type: c5.2xlarge
  beta.kubernetes.io/os: linux
  failure-domain.beta.kubernetes.io/region: us-west-2
```

## Part 5 — Karpenter NodePool configuration

**Check the active NodePool**

```sh
k get nodepools default -o yaml
```

```yaml
spec:
  disruption:
    consolidationPolicy: WhenUnderutilized
    expireAfter: 720h
  limits:
    cpu: 1000
  template:
    spec:
      nodeClassRef:
        name: default
      requirements:
        - key: kubernetes.io/arch
          operator: In
          values:
            - amd64
        - key: karpenter.sh/capacity-type
          operator: In
          values:
            - spot
        - key: karpenter.k8s.aws/instance-category
          operator: In
          values:
            - c
            - m
            - r
        - key: karpenter.k8s.aws/instance-generation
          operator: Gt
          values:
            - "2"
```

This config explains everything Karpenter did in Part 4: `consolidationPolicy: WhenUnderutilized` allows active repacking, `capacity-type: spot` explains why it picked cost-efficient spot capacity, and `instance-category: [c, m, r]` + `instance-generation > 2` explains why `c5.2xlarge` specifically was chosen as a fitting, modern-generation instance within those allowed families.



![[Pasted image 20260827181029.png]]
![[Pasted image 20260827181207.png]]
![[Pasted image 20260827181302.png]]
![[Pasted image 20260827181504.png]]
![[Pasted image 20260827181534.png]]
![[Pasted image 20260827181636.png]]
![[Pasted image 20260827181718.png]]
![[Pasted image 20260827181906.png]]
![[Pasted image 20260827182003.png]]
![[Pasted image 20260827182027.png]]
![[Pasted image 20260827182045.png]]

![[Pasted image 20260827182116.png]]
![[Pasted image 20260827182205.png]]

![[Pasted image 20260827182304.png]]
![[Pasted image 20260827182345.png]]
![[Pasted image 20260827182435.png]]

![[Pasted image 20260827182508.png]]
![[Pasted image 20260827182545.png]]

![[Pasted image 20260827182615.png]]
![[Pasted image 20260827182625.png]]

![[Pasted image 20260827182641.png]]

![[Pasted image 20260827182655.png]]

![[Pasted image 20260827182734.png]]
![[Pasted image 20260827182746.png]]
![[Pasted image 20260827182801.png]]