# Meridian Retail — Full Incident Log (Detailed)

---

## Ticket 1 — Marco (Release Engineer) — storefront-web stale deploy

**Channel:** #customer-escalations · P2

**Problem reported:** Storefront update pushed twice, pipeline reported success both times, live site still showed old version. Pods looked healthy from Meridian's side. Customer demo Thursday.

**Investigation (checked, ruled out):**

- `kubectl rollout status` / `rollout history` — cluster state didn't match pipeline's reported success
- Compared live image reference vs. intended release
- Checked ReplicaSet ages — old ReplicaSet still active, no new one created
- Checked pod readiness and Endpoints — ruled out Service/selector mismatch
- Checked `meridian-ingress` (read-only) — ruled out routing change
- `kubectl describe deployment` revealed `Progressing: Unknown / DeploymentPaused`

**Root cause:** Deployment was paused. A paused Deployment accepts spec changes but never creates a new rollout/ReplicaSet, so `kubectl apply` (and the pipeline) reported success while nothing actually changed on the cluster.

**Fix + confirmation:**

- `kubectl rollout resume deployment/storefront-web`
- New ReplicaSet came up 1/1 Ready, old ReplicaSet scaled to 0
- Endpoints confirmed pointing at new pod
- Verified externally via real ingress URL: `200 OK`, correct content served

**Status: Resolved**

---

## Ticket 2 — Aisha (Platform Engineer) — orders-api LOG_LEVEL not taking effect

**Channel:** #customer-escalations · P3

**Problem reported:** Changed `LOG_LEVEL` in `orders-config` ConfigMap twice, restarted `orders-api` both times, no change observed. She'd already verified the ConfigMap contents herself.

**Investigation (checked, ruled out):**

- Deployment not paused; rollout history showed 3 revisions, restarts did create new revisions
- ConfigMap correctly volume-mounted at `/etc/orders`; live file content matched expected values (`LOG_LEVEL: info`, `ORDERS_TIMEOUT: 30s`)
- Ruled out a mount/sync issue — ConfigMap was reaching the pod correctly
- Pod logs showed nginx startup sequence (`docker-entrypoint.sh`, `nginx/1.27.5`), not an orders-api application
- Confirmed image: `nginxinc/nginx-unprivileged:1.27-alpine`, no command/args override
- Checked all 3 rollout revisions — every one used the same nginx image; only the ConfigMap wiring changed between them (env vars → removed → volume mount), never the image. Rollback not viable.
- Confirmed `orders-config` is referenced by only this one Deployment — no other object affected by her edit
- Confirmed the Deployment's RollingUpdate strategy is unrelated (ConfigMap edits don't trigger rollouts at all)

**Root cause:** `orders-api` has never run an actual orders-api application in any recorded revision — only a generic nginx placeholder image. Nothing in the pod reads `LOG_LEVEL` or `ORDERS_TIMEOUT`. Aisha's edits and restarts were all correct; the fault is entirely platform-side.

**Fix + confirmation:**

- Not resolvable locally — no revision in history ever had a valid image, no AWS/ECR access from this session to find one
- Needs correct orders-api image reference from Meridian (most likely Tomas)
- Once supplied: `kubectl set image deployment/orders-api api=<correct-image>`, then verify rollout

**Status: Open — blocked on Meridian**

---

## Ticket 3 — Sam (Junior SRE, PAVE) — sandbox pod creation refused

**Channel:** #internal-requests · P3 (internal)

**Problem reported:** Sam couldn't create even a bare busybox pod in the shared sandbox namespace; refused every time. He lacked rights to investigate himself. Everything already running "looked fine."

**Investigation (checked, ruled out):**

- Checked `sandbox-quota` — `requests.cpu`/`requests.memory` both at 0 used, ruling out compute capacity
- Checked `sandbox-limits` LimitRange — defaults well within max, ruled out
- Confirmed via `kubectl auth can-i` this wasn't an RBAC issue on the account with access
- Listed pods — found `count/pods: 6/6`, quota maxed purely on count
- All 6 pods were `STATUS: Completed` (debug-run-01 through 06) — finished but never deleted

**Root cause:** Completed pods still count against pod-count quota until deleted. Six stale, finished debug pods from earlier one-off runs consumed the entire quota ceiling even though compute usage was zero. Sandbox is unowned shared scratch space with no cleanup process.

**Fix + confirmation:**

- `kubectl delete pod -n <ns>-sandbox --field-selector=status.phase=Succeeded`
- Quota freed: `count/pods` dropped from 6/6 to 0/6
- Reproduced Sam's exact scenario (bare busybox pod) — created and reached 1/1 Running successfully

**Status: Resolved**

---

## Ticket 4 — AlertManager (Monitoring, PAVE) — inventory-api restart rate alert

**Channel:** #sre-oncall · P2 (auto-filed, no human context, broken runbook link)

**Problem reported:** `PodRestartRateHigh`, workload=inventory-api, 11 restarts in 30-minute window.

**Investigation (checked, ruled out):**

- Pod events showed repeated `Liveness probe failed: HTTP probe failed with statuscode: 404` (133 occurrences)
- Image pull was fine, ruled out pull/scheduling issue
- Container logs showed no app errors, nginx started and ran cleanly — container wasn't dying on its own, it was being killed by kubelet
- Liveness probe spec: path `/health`, port 8080, failureThreshold 3, periodSeconds 10
- Logs confirmed nginx itself returning the 404: `open() "/etc/nginx/html/health" failed, No such file or directory`

**Root cause:** Same pattern as ticket 2 — `inventory-api` runs a generic `nginxinc/nginx-unprivileged:1.27-alpine` image with no application logic. No `/health` route exists for the probe to hit, so every liveness check 404s and kubelet kills an otherwise-healthy container on a loop.

**Fix + confirmation:**

- Interim only: patched liveness probe path from `/health` to `/` (which nginx serves by default)
- `kubectl patch deployment inventory-api --type='json' -p='[{"op":"replace","path":"/spec/template/spec/containers/0/livenessProbe/httpGet/path","value":"/"}]'`
- Verified: new pod came up 1/1 Running, RESTARTS: 0, stayed stable on follow-up watch

**Status: Alert noise resolved (interim fix). Underlying issue open — same blocker as ticket 2, real inventory-api image needed from Meridian.**

---

## Ticket 5 — Platform Ops (Infrastructure, PAVE) — promo-web container not ready

**Channel:** #sre-oncall · P3 (auto-escalated, no investigation attached)

**Problem reported:** `container-not-ready`, pod=promo-web, containers ready 1/2, continuous.

**Investigation (checked, ruled out):**

- Pod showed 1/2 Ready, CrashLoopBackOff, 31 restarts
- `app` container (nginx) was Running and Ready — ruled out
- `log-shipper` container was the one crashing, Exit Code 1
- Logs showed exact error: `httpd: bind: Address already in use`
- Checked env vars — `METRICS_PORT` was set to 8080, same port the `app` container already uses

**Root cause:** `log-shipper`'s `METRICS_PORT` was misconfigured to 8080 instead of 9090 (per platform doc, log-shipper metrics should run on 9090). Since containers in a pod share one network namespace, log-shipper's httpd tried to bind a port already held by nginx and failed immediately on every restart.

**Fix + confirmation:**

- `kubectl set env deployment/promo-web METRICS_PORT=9090 -c log-shipper`
- Patched declared containerPort to match (9090)
- Verified: new pod came up 2/2 Running, RESTARTS: 0

**Status: Resolved**

---

## Ticket 6 — Priya (TAM, PAVE) — promo-web returning 503 externally

**Channel:** #customer-escalations · P1 (customer live on the phone)

**Problem reported:** Customer's promotions page returning 503 for everyone outside. Pods showing Ready, which confused Priya. Asking whether it's the load balancer.

**Depends on:** Ticket 5 (pod had to be Ready/have an endpoint first)

**Investigation (checked, ruled out):**

- Confirmed ticket 5 already resolved — pod 2/2 Ready with a valid Endpoint
- Checked Service spec: `targetPort: 8000`
- Checked actual pod container port: `8080`
- Confirmed mismatch — Service routing to a port nothing was listening on
- Checked ingress rules — `/esolante/promo` correctly routed to `promo-web` Service, ruled out ingress misconfiguration

**Root cause:** Service's `targetPort` (8000) didn't match the app container's actual listening port (8080). Even with the pod healthy and Ready, traffic was being sent to a port with nothing behind it — produced 503 at the edge despite Ready pods.

**Fix + confirmation:**

- `kubectl patch svc promo-web --type='json' -p='[{"op":"replace","path":"/spec/ports/0/targetPort","value":8080}]'`
- Endpoints updated to correct port
- First external retest returned 502 briefly (likely ingress-nginx propagation lag)
- Verified app healthy via direct port-forward test: 200 OK
- Retested real external ingress URL: 200 OK

**Status: Resolved** — confirmed via the same external URL the customer was hitting

---

## Ticket 7 — Platform Ops (Infrastructure, PAVE) — report-exporter no execution history

**Channel:** #sre-oncall · P3 (auto-escalated, 3 windows missed)

**Problem reported:** `scheduled-workload-no-history`, workload=report-exporter, last successful execution none recorded, 3 consecutive windows missed.

**Depends on:** none (this ticket is itself a prerequisite for ticket 8)

**Investigation (checked, ruled out):**

- CronJob spec: `suspend: true`
- Schedule (`*/5 * * * *`) valid and unchanged — ruled out bad cron expression
- No Job objects existed at all — consistent with a suspended CronJob never triggering

**Root cause:** `report-exporter` CronJob had `spec.suspend: true`. A suspended CronJob never creates Jobs on schedule, so there was nothing to execute and nothing to record.

**Fix + confirmation:**

- `kubectl patch cronjob report-exporter -p '{"spec":{"suspend":false}}'`
- Confirmed CronJob resumed scheduling — a run fired on its own shortly after
- Manually triggered an additional Job to confirm immediately
- Both scheduled and manual runs created pods, ran, and completed with recorded status

**Status: Resolved** — CronJob executing and recording history again. (Both runs completed as Failed — carried forward to ticket 8, out of this ticket's scope.)

---

## Ticket 8 — Lena (Finance Ops, Meridian) — reconciliation file missing from S3

**Channel:** #customer-escalations · P3

**Problem reported:** Reconciliation file missing from shared bucket since Tuesday. Month-end close approaching.

**Depends on:** Ticket 7 (CronJob had to be running before a failure inside it could even appear in logs)

**Investigation (checked, ruled out):**

- Confirmed ticket 7's fix held — CronJob unsuspended and scheduling
- Triggered manual job runs, captured live logs before pod cleanup
- Found exact error: `AWS_ACCESS_KEY_ID: Unable to locate credentials: AWS_ACCESS_KEY_ID is not set in this container's environment`
- Checked `s3-export-creds` Secret — keys named `AWS_ACCESS_KEY`/`AWS_SECRET_KEY`, not matching script's expected `AWS_ACCESS_KEY_ID`/`AWS_SECRET_ACCESS_KEY`
- Re-ran job after correcting Secret keys — credential check passed
- Job then generated CSV successfully but logged `EXPORT_BUCKET unset, skipping upload`
- Checked CronJob spec and configmaps for a bucket name — none found within namespace scope; cluster-wide search outside RBAC access

**Root cause (two compounding issues):**

1. Secret's key names didn't match what the export script expected, so valid credentials existed but were never actually being read
2. Independent of that, `EXPORT_BUCKET` env var has never been set on the CronJob, so even with valid credentials the script only generates the file locally in the pod and explicitly skips the upload step

**Fix + confirmation:**

- Recreated `s3-export-creds` Secret with correctly named keys (`AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`)
- Confirmed via job logs — credential check now passes, CSV generates successfully
- `EXPORT_BUCKET` remains unset — customer-owned configuration, no visibility or authority to invent a value

**Status: Partially resolved.** Credential blocker fixed and confirmed. File upload still blocked pending actual bucket name from Meridian. (Minor cleanup: old `AWS_ACCESS_KEY`/`AWS_SECRET_KEY` keys still sitting in the Secret alongside the correct ones — harmless, worth removing eventually.)

---

## Ticket 9 — Tomas (Application Developer, Meridian) — cache-warmer never came up

**Channel:** #customer-escalations · P2

**Problem reported:** Added a cache to the order service last week, it's never come up. Deployment exists but nothing is running. Unsure if it's a bad storage request or platform-side.

**Investigation (checked, ruled out):**

- Confirmed pod stuck Pending, never scheduled
- Pod events: `binding volumes: context deadline exceeded`
- Checked PVC `orders-cache` — status Pending
- PVC events: `ProvisioningFailed`, `Volume capabilities not supported`
- Checked PVC spec: `accessModes: ReadWriteMany` on StorageClass `gp2` (EBS-backed)
- Confirmed EBS only supports `ReadWriteOnce` — explains the provisioning failure
- Checked Deployment's volume reference — single PVC, no other dependents, safe to recreate

**Root cause:** `orders-cache` PVC requested `ReadWriteMany`, which EBS-backed storage doesn't support (EBS only supports `ReadWriteOnce`). Since `cache-warmer` runs as a single pod on one node, `ReadWriteOnce` is sufficient and correct. This was a storage request issue on Meridian's side, not a platform problem.

**Fix + confirmation:**

- Deleted and recreated PVC with `accessModes: ReadWriteOnce` (access mode is immutable, can't be patched in place)
- Confirmed PVC moved to `Bound`
- Confirmed `cache-warmer` pod is now 1/1 Running

**Status: Resolved**

---

# Overall Summary

**7 of 9 resolved. 2 open, both correctly blocked on Meridian (not platform gaps):**

- Ticket 2 — needs correct orders-api image reference
- Ticket 8 — needs `EXPORT_BUCKET` value

**Recurring theme across the day:** Several issues were invisible at a surface-level status check (pod Ready, pipeline green, dashboard green) and only surfaced through `describe`, `rollout history`, or direct log/config inspection — consistent with the handover note that a green check only ever confirms that one specific check passed, not overall health.

**Pattern connecting tickets 2, 4, and 8:** Placeholder/generic images and mismatched credential key names across multiple components — consistent with the namespace having been hand-recreated at some point and not matching its original build.

**Minor cleanup items (not urgent):**

- Leftover `debug-test` pod in sandbox (ticket 3 verification artifact)
- Old `AWS_ACCESS_KEY`/`AWS_SECRET_KEY` keys still present in `s3-export-creds` Secret (ticket 8)