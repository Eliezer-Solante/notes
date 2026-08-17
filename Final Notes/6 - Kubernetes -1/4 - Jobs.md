# Creating a Job in Kubernetes

A Kubernetes **Job** runs one or more pods until they complete successfully — good for batch tasks, one-off scripts, or migrations (as opposed to a Deployment, which keeps pods running indefinitely).

## 1. Basic YAML Definition

Create a file, e.g. `job.yaml`:

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: my-job
spec:
  template:
    spec:
      containers:
      - name: my-job-container
        image: busybox:latest
        command: ["echo", "Hello from the Job!"]
      restartPolicy: Never
  backoffLimit: 4
```

Key fields:

- **`restartPolicy`**: must be `Never` or `OnFailure` (not `Always`) for Jobs.
- **`backoffLimit`**: how many times Kubernetes retries the Job before marking it failed (default 6).

## 2. Apply It

```bash
kubectl apply -f job.yaml
```

## 3. Check Status

```bash
kubectl get jobs
kubectl describe job my-job
kubectl get pods --selector=job-name=my-job
```

## 4. View Logs

```bash
kubectl logs -l job-name=my-job
```

## 5. Optional Variations

**Run multiple pods in parallel:**

```yaml
spec:
  parallelism: 3
  completions: 6
```

**Set a time limit:**

```yaml
spec:
  activeDeadlineSeconds: 300
```

**Imperative one-liner (quick testing, no YAML file):**

```bash
kubectl create job my-job --image=busybox -- echo "Hello from the Job!"
```

**Run a Job on a schedule** (use a `CronJob` instead):

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
          - name: my-job-container
            image: busybox:latest
            command: ["echo", "Hello from the CronJob!"]
          restartPolicy: OnFailure
```

## 6. Clean Up

```bash
kubectl delete job my-job
```

Want me to tailor this to a specific use case (e.g. a database migration, a batch processing script, or running it with a ConfigMap/Secret)?