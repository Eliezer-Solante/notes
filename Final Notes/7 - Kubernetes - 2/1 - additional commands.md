Fast Reference Guide

- **Interactive Terminal**: `kubectl exec -it <pod-name> -- /bin/sh`
- **Single Command Execution**: `kubectl exec <pod-name> -- ls -la /cache`

---

Step-by-Step Instructions

1. Find Your Pod Name

List the active Pods in your namespace to copy the correct target name:
```bash
kubectl get pods
```
Use code with caution.

2. Execute a Single Command

To run a command and view its output instantly without entering the container, use this structure:
```bash
kubectl exec <pod-name> -- <command>
```
Use code with caution.

_(Example: `kubectl exec emptydir-shared-pod -- cat /cache/data.txt`)_

3. Start an Interactive Shell Session (Most Common)

To stay inside the container and run multiple commands, pass the `-i` (stdin) and `-t` (tty) flags, followed by a shell binary like `sh` or `bash`:
```bash
kubectl exec -it <pod-name> -- /bin/sh
```
Use code with caution.

If `/bin/sh` is not found, try `/bin/bash` or `sh`.

---

Handling Multi-Container Pods

If your Pod contains more than one container (such as a primary app and a sidecar logging container), you **must specify the target container** using the `-c` flag. If skipped, Kubernetes will default to the very first container defined in the manifest.
```bash
kubectl exec -it <pod-name> -c <container-name> -- /bin/sh
```
Use code with caution.

**Practical Example (Using the `emptyDir` template from earlier):**

- Access the writer: `kubectl exec -it emptydir-shared-pod -c writer-app -- /bin/sh`
- Access the reader: `kubectl exec -it emptydir-shared-pod -c reader-app -- /bin/sh`

---

Handling Alternative Namespaces

If your Pod resides outside the `default` namespace, append the namespace flag:
```bash
kubectl exec -it <pod-name> -n <namespace-name> -- /bin/sh
```
Use code with caution.

---