**Basic CronJob manifest**

yaml

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: my-cronjob
spec:
  schedule: "*/5 * * * *"        # standard cron syntax
  concurrencyPolicy: Forbid      # Allow | Forbid | Replace
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 1
  startingDeadlineSeconds: 60
  jobTemplate:
    spec:
      backoffLimit: 2
      template:
        spec:
          containers:
          - name: my-job
            image: my-image:latest
            command: ["/bin/sh", "-c", "echo hello; date"]
          restartPolicy: OnFailure
```


Apply it:

```bash
kubectl apply -f cronjob.yaml
```

Here's a minimal working example — a job that runs every 5 minutes and prints a message:

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: hello-cronjob
spec:
  schedule: "*/5 * * * *"
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: hello
            image: busybox
            command: ["/bin/sh", "-c", "echo Hello from Kubernetes CronJob at $(date)"]
          restartPolicy: OnFailure
```

Apply and check it:

```bash
kubectl apply -f hello-cronjob.yaml
kubectl get cronjobs
kubectl get jobs --watch      # see it spawn a Job every 5 min
kubectl logs job/<job-name>   # check output
```

That's it — no extra fields needed to get something running. Adjust the `schedule`, `image`, and `command` for your actual task.

**Schedule syntax** (standard 5-field cron)

```
┌───────────── minute (0-59)
│ ┌───────────── hour (0-23)
│ │ ┌───────────── day of month (1-31)
│ │ │ ┌───────────── month (1-12)
│ │ │ │ ┌───────────── day of week (0-6, Sun=0)
│ │ │ │ │
* * * * *
```

Examples: `0 2 * * *` (daily 2am), `*/15 * * * *` (every 15 min), `0 0 * * 0` (weekly Sunday midnight).

**Key fields to know**

- `concurrencyPolicy`: `Forbid` skips a new run if the previous is still going; `Replace` kills the old one and starts new; `Allow` (default) lets them overlap.
- `startingDeadlineSeconds`: how late a missed run can still start (e.g. after a control-plane outage) before being skipped.
- `successfulJobsHistoryLimit` / `failedJobsHistoryLimit`: how many completed Job objects to retain for debugging/logs.
- `.spec.jobTemplate.spec.backoffLimit`: retries for a failed Job run.
- Timezone: as of Kubernetes 1.27+, you can set `.spec.timeZone: "America/New_York"` (IANA name) instead of relying on cluster-local time (usually UTC).

**Useful commands**

bash

```bash
kubectl get cronjobs
kubectl get jobs --watch                     # jobs spawned by cronjobs
kubectl create job --from=cronjob/my-cronjob manual-run-1   # trigger manually, on-demand
kubectl delete cronjob my-cronjob
kubectl patch cronjob my-cronjob -p '{"spec":{"suspend":true}}'   # pause without deleting
```

**Common gotchas**

- Missed schedules (cluster down, controller-manager restart) beyond `startingDeadlineSeconds` are just skipped, not queued.
- Jobs left behind by `successfulJobsHistoryLimit`/`failedJobsHistoryLimit` still count against etcd/API server load if you leave a huge history.
- If your job needs to reach other cluster resources, don't forget `serviceAccountName` and appropriate RBAC in the pod template.
- Container image and command failures show up in the Job/Pod status, not the CronJob object itself — check `kubectl describe job <job-name>` when a run doesn't behave as expected.