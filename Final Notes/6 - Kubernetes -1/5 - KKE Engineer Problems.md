
# Nginx + PHP-FPM Kubernetes Troubleshooting Notes

## Issue

Pod `nginx-phpfpm` (ConfigMap: `nginx-config`) was not serving the website correctly.

## Root Cause

The `shared-files` volume (`emptyDir`) was mounted at **different paths** in the nginx container and the PHP-FPM container. Because the mount paths didn't match, the two containers weren't actually sharing the same directory — files placed in nginx's document root weren't visible to PHP-FPM, and vice versa.

```yaml
# Problem: mountPath mismatch across containers
volumeMounts:
  - mountPath: /usr/share/nginx/html   # nginx-container
    name: shared-files
  - mountPath: /var/www/html           # php-fpm-container (different path!)
    name: shared-files
```

Meanwhile, the `nginx-config` ConfigMap's `root` directive pointed to yet another path (`/var/www/html`), adding to the confusion — but changing only the ConfigMap would **not** have been a complete fix, since it doesn't address the container-to-container mount mismatch.

## Fix

1. **Align the `volumeMounts` in both containers** so `shared-files` is mounted at the **same `mountPath`** in both the nginx and PHP-FPM containers (e.g. both at `/var/www/html`).
2. Ensure the nginx ConfigMap's `root` directive matches that same shared path.
3. Apply the updated pod/deployment spec.
4. Copy `index.php` into the now-correctly-shared volume via the nginx container:
    
    ```bash
    kubectl cp /home/thor/index.php nginx-phpfpm:/var/www/html/index.php -c nginx-container
    ```
    
5. Verify the site loads via the **Website** button.

## Key Lesson

When two containers in a pod need to share files (e.g. nginx serving static content + PHP-FPM processing scripts), **both containers must mount the shared volume at the identical path**. A mismatched `mountPath` between containers is a common but easy-to-miss cause of "file not found" or empty document root issues — checking the ConfigMap alone isn't enough; always cross-check `volumeMounts` across _all_ containers in the pod spec.

## Useful Diagnostic Commands

```bash
# Inspect full pod spec (volumes + volumeMounts per container)
k get pod nginx-phpfpm -o yaml

# Check container logs
k logs nginx-phpfpm -c nginx-container
k logs nginx-phpfpm -c <php-fpm-container-name>

# Reload nginx config without restarting the pod
k exec nginx-phpfpm -c nginx-container -- nginx -s reload

# Copy a file into a specific container in a multi-container pod
kubectl cp <local-path> <pod-name>:<container-path> -c <container-name>
```


Here's the complete solution covering all four parts. Since you need both a Pod and a ConfigMap, that's two manifests (or one file with `---` separating them).

**1. ConfigMap (`time-config.yaml`)**

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: time-config
  namespace: nautilus
data:
  TIME_FREQ: "5"
```

**2. Pod (`time-check.yaml`)**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: time-check
  namespace: nautilus
spec:
  containers:
    - name: time-check
      image: busybox:latest
      command: ["/bin/sh", "-c"]
      args:
        - "while true; do date; sleep $TIME_FREQ; done >> /opt/dba/time/time-check.log"
      env:
        - name: TIME_FREQ
          valueFrom:
            configMapKeyRef:
              name: time-config
              key: TIME_FREQ
      volumeMounts:
        - name: log-volume
          mountPath: /opt/dba/time
  volumes:
    - name: log-volume
      emptyDir: {}
```

**How each part maps:**

|Requirement|Where it's handled|
|---|---|
|Namespace `nautilus`|`metadata.namespace` in both manifests|
|Pod `time-check`, container `time-check`, image `busybox:latest`|Pod spec, `containers[0]`|
|ConfigMap `time-config` with `TIME_FREQ=5`|ConfigMap manifest|
|Command with the while loop|`command` + `args` using `sh -c`|
|Output written to `/opt/dba/time/time-check.log`|`>>` redirect at the end of `args`, landing inside the mounted volume|
|Env var `TIME_FREQ` from the ConfigMap key|`env[0].valueFrom.configMapKeyRef`|
|Volume `log-volume` mounted at `/opt/dba/time`|`volumes` + `volumeMounts`|

A note on the volume type: I used `emptyDir` since the task doesn't specify a particular storage backend (hostPath, PVC, etc.) — it just says "create a volume." `emptyDir` is the simplest choice that satisfies "create a volume and mount it," but if your actual environment expects a specific volume type (e.g., a PVC already provisioned), swap that section accordingly.

**To deploy (from the jump host, since `kubectl` is already configured there):**

```bash
kubectl apply -f time-config.yaml
kubectl apply -f time-check.yaml
```

**To verify it's working:**

```bash
kubectl get pods -n nautilus
kubectl exec -n nautilus time-check -- cat /opt/dba/time/time-check.log
```

You should see timestamps appearing roughly every 5 seconds as the log accumulates.