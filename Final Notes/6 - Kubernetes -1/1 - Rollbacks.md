
# Kubernetes Deployment Rollback — Notes

## 1. Check rollout history

bash

```bash
kubectl rollout history deployment/<name>
```

- Shows revision numbers.
- If CHANGE-CAUSE shows "N/A", revisions weren't annotated. Fix going forward with `kubernetes.io/change-cause` annotations (or `--record`, deprecated but often works).

View details of a specific revision:

bash

```bash
kubectl rollout history deployment/<name> --revision=<N>
```

## 2. Roll back

bash

```bash
# to previous revision
kubectl rollout undo deployment/<name>

# to a specific revision
kubectl rollout undo deployment/<name> --to-revision=<N>
```

## 3. Watch progress

bash

```bash
kubectl rollout status deployment/<name>
```

Blocks until complete/fails — useful in scripts/CI.

## 4. Verify

bash

```bash
kubectl get pods -l app=<label>
kubectl describe deployment/<name>
```

Check image tags, replica counts, pod readiness.

## Gotchas

- **Revision history limit**: default keeps only 10 old ReplicaSets (`spec.revisionHistoryLimit`). Older ones may be garbage collected. Check with `kubectl get rs -l app=<label>`.
- **Other resources aren't reverted**: ConfigMaps/Secrets/CRDs changed alongside the deployment need to be rolled back separately (unless managed via GitOps/Helm for atomic changes).
- **Helm-managed deployments**: use `helm rollback <release> <revision>` instead of `kubectl rollout undo` to avoid state drift.
- **StatefulSets**: same undo command, but watch ordering/PVC behavior more closely.
- **If rollback pods don't become ready**: check `kubectl describe pod` — old image may have been pruned from the registry or node.