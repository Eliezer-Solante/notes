
![[Pasted image 20260817161432.png]]


![[Pasted image 20260817161457.png]]


# Metrics Server in Kubernetes

## What It Is

**metrics-server** is a lightweight, cluster-wide aggregator of resource usage data (CPU and memory). It's a standalone component you deploy on top of Kubernetes — it doesn't come installed by default in most clusters (though many managed services like GKE, EKS, and AKS include it or make it easy to add).

## How It Works

1. It runs as a Deployment inside your cluster (usually in `kube-system`).
2. It collects resource usage stats from the **kubelet** on every node — specifically from a component inside the kubelet called **cAdvisor**, which tracks per-container CPU/memory usage.
3. It aggregates this data and exposes it through the **Kubernetes API server** via the **Metrics API** (`metrics.k8s.io`), rather than through a separate endpoint you have to query directly.
4. It polls at a set interval (default every **15 seconds** — configurable via `--metric-resolution`), and only keeps the **most recent** data point. There is no historical storage.

```
kubelet (cAdvisor) → metrics-server → Metrics API (metrics.k8s.io) → consumers
```

## What It Gives You

### `kubectl top`

The most direct, hands-on use — quick real-time visibility without a dashboard.

```bash
kubectl top nodes
kubectl top pods
kubectl top pods -n my-namespace --containers
```

### Horizontal Pod Autoscaler (HPA)

HPA relies on metrics-server to read current CPU/memory usage and decide whether to scale replicas up or down.

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: my-app-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: my-app
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

Without metrics-server running, this HPA would sit stuck, unable to compute the target and unable to scale.

### Vertical Pod Autoscaler (VPA)

Similarly relies on this same data to recommend or automatically adjust `requests`/`limits` for containers.

## What It Is NOT

This is the most important thing to understand about metrics-server — its scope is intentionally narrow:

|metrics-server|Prometheus|
|---|---|
|CPU & memory only|Any metric type (custom, business, app-level)|
|Real-time snapshot only|Full historical time-series|
|No alerting|Alertmanager integration|
|No dashboards|Grafana integration|
|Minimal resource footprint|Heavier, more components|
|Built for autoscaling decisions|Built for observability/debugging|

It's **not** a general-purpose monitoring solution — you can't graph trends, set alerts, or track anything beyond CPU/RAM with it. If a Pod died an hour ago, metrics-server has zero record of what its resource usage looked like before that.

## Installing It

Typically deployed via the official manifest or Helm:

```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

## A Common Gotcha

On some clusters (particularly local/dev setups like Minikube, kind, or self-signed kubelet certs), metrics-server fails to start because it can't verify kubelet TLS certificates. The usual fix is adding this flag to its Deployment args:

```yaml
- --kubelet-insecure-tls
```

This is fine for local dev/testing but **should not be used in production**, since it disables certificate verification between metrics-server and the kubelets.

## Quick Health Check

If `kubectl top` returns nothing or errors like `error: Metrics API not available`, that's the first sign metrics-server either isn't installed or is crash-looping — worth checking with:

```bash
kubectl get deployment metrics-server -n kube-system
kubectl logs -n kube-system deploy/metrics-server
```