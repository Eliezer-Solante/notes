
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