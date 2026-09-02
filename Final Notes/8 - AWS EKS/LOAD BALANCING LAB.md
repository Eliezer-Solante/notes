# Lab notes: AWS Load Balancer Controller on EKS

## Overview

This lab covers three integration patterns for the AWS Load Balancer Controller on an EKS cluster:

1. Install the controller itself
2. Expose a Deployment via `Service type: LoadBalancer` (creates an NLB)
3. Expose a Deployment via `Ingress` (creates an ALB)

---

## Part 1 — Install the AWS Load Balancer Controller

**Location of the manifest:**

```
/root/amazon-elastic-kubernetes-service-course/eks/v2_7_2_full.yaml
```

**Step 1 — Update the cluster name in the manifest**

Find the controller's Deployment spec and set `--cluster-name` to match your actual EKS cluster:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: aws-load-balancer-controller
  namespace: kube-system
spec:
  template:
    spec:
      containers:
        - args:
            - --cluster-name=demo-eks   # must match your real cluster name
```

> Note: this controller must know which cluster it's managing — it uses this to tag and discover AWS resources (subnets, security groups) tied to your cluster's VPC.

**Step 2 — Apply the manifest**

```sh
kubectl apply -f v2_7_2_full.yaml
```

This creates the controller Deployment plus all supporting resources it needs (CRDs, RBAC roles, webhooks, service account).

At this point, the controller is running and **watching the cluster** for `Service type: LoadBalancer` and `Ingress` resources — but it hasn't created anything in AWS yet. It only acts once you create workloads that reference it.

---

## Part 2 — Expose an app via `Service type: LoadBalancer` (→ NLB)

**Deployment (`webapp-color`)**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp-color
  labels:
    app: webapp-color
spec:
  replicas: 3
  selector:
    matchLabels:
      app: webapp-color
  template:
    metadata:
      labels:
        app: webapp-color
    spec:
      containers:
      - name: webapp-color
        image: kodekloud/webapp-color
        ports:
        - containerPort: 8080
```

**Service (`webapp-color`)**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: webapp-color
  namespace: default
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-type: "nlb"
    service.beta.kubernetes.io/aws-load-balancer-internal: "false"
spec:
  type: LoadBalancer
  selector:
    app: webapp-color
  ports:
    - protocol: TCP
      port: 80
      targetPort: 8080
```

**Key annotations explained**

|Annotation|Purpose|
|---|---|
|`aws-load-balancer-type: "nlb"`|Tells the controller to provision a **Network Load Balancer** (L4/TCP) instead of the older "classic" ELB default|
|`aws-load-balancer-internal: "false"`|Makes the NLB **internet-facing** (publicly reachable). Setting this to `"true"` would create an internal-only NLB, reachable only from inside the VPC|

**Apply both manifests**

```sh
kubectl create -f <deployment-file>.yaml
kubectl create -f <service-file>.yaml
```

**Verify in AWS**

1. Go to the **EC2 console → Load Balancers**.
2. Find the newly created NLB (provisioned automatically by the controller because of `type: LoadBalancer`).
3. Copy its **DNS name** and open it in a browser.

Expected output:

```
Hello from webapp-color-564cb8d898-j2shn!
```

The pod name in the response confirms which of the 3 replicas actually served the request — proof the NLB is load-balancing across the Deployment's Pods.

### Quiz check

**Q: What annotations are used in the `webapp-color` Service?**

✅ **A.** `service.beta.kubernetes.io/aws-load-balancer-type: "nlb"` ✅ **B.** `service.beta.kubernetes.io/aws-load-balancer-internal: "false"`

❌ C. `aws-load-balancer-type: "alb"` — wrong; ALB is only used with `Ingress`, not `Service type: LoadBalancer`, in this lab. ❌ D. `aws-load-balancer-internal: "true"` — wrong; the lab explicitly sets this to `"false"` to keep the load balancer internet-facing.

---

## Part 3 — Expose an app via `Ingress` (→ ALB)

This is the second integration pattern the AWS Load Balancer Controller supports: watching `Ingress` resources and provisioning an **Application Load Balancer (ALB)** instead of an NLB.

**Manifest location:**

```
/root/amazon-elastic-kubernetes-service-course/eks/2048_full.yaml
```

**Create the resources**

```sh
kubectl create -f 2048_full.yaml
```

This manifest bundles a Deployment, Service, and an `Ingress` object together (a common pattern for demo apps like `game-2048`).

**Check the Ingress**

```sh
kubectl get ingress -n game-2048
```

Output shows an auto-generated ALB address, e.g.:

```
k8s-game2048-ingress2-46dbc758ae-232597482.us-east-1.elb.amazonaws.com
```

Open that address in a browser to reach the app.

---

## Comparing the two patterns from this lab

||`Service type: LoadBalancer`|`Ingress`|
|---|---|---|
|**AWS resource created**|NLB (Layer 4 / TCP)|ALB (Layer 7 / HTTP)|
|**Routing capability**|None — just forwards raw TCP to Pods|Host/path-based routing, one ALB can front multiple apps|
|**Typical use case**|Single app, TCP-level exposure, non-HTTP protocols|Multiple apps behind one load balancer, HTTP-aware routing|
|**Annotation-driven config**|`aws-load-balancer-type`, `aws-load-balancer-internal`|`alb.ingress.kubernetes.io/*` annotations (scheme, target-type, etc.)|

---

## How this ties into the broader model

```
kubectl apply -f v2_7_2_full.yaml
        ↓
AWS Load Balancer Controller pod starts running (watches API)
        ↓
   ┌───────────────────────────┬───────────────────────────┐
   │                           │                           │
Service (type: LoadBalancer)  Ingress
   │                           │
   ▼                           ▼
Controller provisions NLB   Controller provisions ALB
   │                           │
   ▼                           ▼
Traffic → NLB → Pod         Traffic → ALB → Pod
(direct L4 forwarding)      (L7 routing via host/path rules)
```

In both cases, the **controller pod itself never sits in the traffic path** — its only job is to watch Kubernetes objects and keep the real AWS load balancer's configuration in sync. Once provisioned, the actual ALB/NLB (AWS-managed infrastructure) handles all live traffic on its own.