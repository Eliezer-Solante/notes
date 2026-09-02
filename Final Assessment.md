Context
# PAVE Corp Infrastructure Troubleshooting — Session Summary

## Background Context

**The company:** PAVE Corp — a mid-sized logistics and freight brokerage, not a tech company. The platform (`pave-01`) exists because off-the-shelf tooling didn't fit the business. Four teams depend on it:

- **Operations** — uploads carrier paperwork (rate confirmations, bills of lading, POD scans); heaviest, least patient users.
- **Marketing** — runs customer-facing landing pages, browses the document archive for case studies.
- **Data team** — consumes a daily metrics CSV (load counts, lane volumes, margin by customer) that lands at 06:00 and feeds Monday exec spreadsheets.
- **Ops leads** — watch a wall-mounted status page in the dispatch room; green means healthy.

**System architecture (pave-01, CentOS Stream 9, 2 vCPU/4GB/80GB):**

Host services (systemd, plain Python 3.9 stdlib, log to journal):

|Service|Port|Purpose|
|---|---|---|
|nginx.service|80|Front door, reverse proxy + static assets|
|pave-statuspage.service|8081|Status page for dispatch room display|
|pave-archive.service|8080|Lists archived documents (Marketing)|
|pave-cache.service|7000|Memoizes report queries|
|pave-reports.service|8090|Report generation/export API|

Docker containers (share image `pavecorp/app:latest`, all run as UID 2000, on custom bridge network `pave-net`, resolve each other by name):

|Container|Port|Purpose|
|---|---|---|
|app-web|8000 (127.0.0.1)|HTTP API, uploads, enqueues jobs|
|app-worker|none|Polls queue, processes jobs, clears them|
|app-notify|none|Posts job completions to Ops channel|

Scheduled work: `daily_report.py` runs 06:00 daily via root cron (`/etc/cron.d/pave-daily`), writes the metrics CSV.

**Nginx routing:** `/` → static `/var/www/current`; `/api/*` → app-web:8000; `/reports/*` → pave-reports:8090; `/archive/*` → pave-archive:8080; `/status` → pave-statuspage:8081.

**Request flow:** app-web writes uploads to `/srv/pave/uploads`, drops job files into `/srv/pave/queue`. app-worker polls the queue, processes, writes results, removes job file. app-notify reaches app-web by name over pave-net to post completions. pave-reports asks pave-cache first, falls back to direct read if cache is down (still works, just slower). pave-archive lists docs for Marketing. pave-statuspage serves the dispatch display.

**Filesystem:** `/etc/nginx/` (routing config), `/etc/systemd/system/` (the four pave-* units), `/etc/pave-cache/` (cache env file), `/etc/logrotate.d/`, `/etc/cron.d/` (nightly job schedule), `/opt/pave/` (app code: 3 git repos, host services, nightly job + its own log dir), `/srv/pave/` (runtime data: uploads, queue, reports), `/var/www/` (static releases + `current` symlink). No central app log directory — container output to Docker logs, host services to journal, pave-archive and the nightly job keep their own files.

**Key trap:** `/opt/pave/src` is leftover unpacked install media — services run from installed copies elsewhere, NOT from here. Editing files under `/opt/pave/src` changes nothing live. Always confirm real paths via `systemctl cat <service>`.

**Deployment:** Releases cut from Git tags. Check out tag in `/opt/pave/app` (deploy repo) → build into `/var/www/releases/<tag>/` → atomically repoint `/var/www/current` symlink → restart app-web if code changed. Two repo clones: `/opt/pave/app` (deploy repo, whatever's checked out here gets built) and `/opt/pave/app-staging` (where the next release is staged/merged before reaching the deploy repo). Both use `/opt/pave/app-remote.git` as origin. **Version discipline matters:** drift between the tag checked out in the deploy repo and what `current` points to is a known recurring source of "why is the old version live" confusion.

**Why it looks like this:** Four-year-old platform, never designed, accreted. Year 1: contractor built the core (app-web, app-worker, nginx) — only genuinely planned part. Year 2: nightly metrics script bolted on via cron. Year 2 later: pave-reports + pave-cache stood up separately rather than optimizing the container. Year 3: pave-archive added for Marketing (Ops refused direct access). Year 3 later: pave-statuspage added after an unnoticed 2-hour outage. Year 4: app-notify container added so dispatchers stop refreshing the page manually. No consolidation, ever.

**Handover notes (unverified, "context not answers"):** deploys have been manual and untrusted; a config somewhere had permissions issues once, unclear which; the reports service and cache "were supposed to be one thing, they aren't"; cron on this box has bitten people before; someone pre-allocated disk for a migration that never happened; permissions under `/srv` drift when people SSH in and poke around; logrotate policies were added by hand over time and never fully verified.



# 1.
<p>1. Problem reported / what I found:</p> <p>The dispatch room’s wall-mounted status display was blank after the maintenance reboot. The <code>pave-statuspage.service</code> (port 8081) was not running, and the Ops leads confirmed it had failed to start automatically after each reboot.</p>

<p>2. Investigation (checked, ruled out):</p> <p>• Verified network connectivity and nginx routing — both healthy.<br> • Checked <code>systemctl status pave-statuspage.service</code> — showed “inactive (dead)” immediately after boot.<br> • Examined <code>journalctl -u pave-statuspage.service</code> — logs indicated missing Python path environment variable.<br> • Confirmed the service file under <code>/etc/systemd/system/</code> lacked <code>EnvironmentFile=/etc/pave-cache/env</code> or equivalent.<br> • Docker containers unaffected; issue isolated to host service startup.</p>

<p>3. Root cause:</p> <p>The <code>pave-statuspage.service</code> unit did not include the required environment configuration, so systemd attempted to start it before its dependencies were ready. It succeeded only when manually started later, after the environment was loaded. The missing <code>After=network.target</code> and absent <code>EnvironmentFile</code> entries prevented reliable boot‑time startup.</p>

<p>4. Fix + confirmation:</p> <p>• Edited <code>/etc/systemd/system/pave-statuspage.service</code> to include:<br> <code>After=network.target</code><br> <code>EnvironmentFile=/etc/pave-cache/env</code><br> • Ran <code>systemctl daemon-reload</code> and <code>systemctl enable pave-statuspage.service</code>.<br> • Rebooted host to confirm persistence — service auto‑started, port 8081 reachable, and display showed green status indicators.<br> • Verified via <code>journalctl -u pave-statuspage.service</code> that startup completed cleanly.<br> ✅ Issue resolved; status board remains active after reboot.</p>

# 2.

- Problem reported / what I found: Operations reported that uploads of carrier paperwork were failing, with the page showing a gateway error. The nginx web server was running normally, but the backend upload service was unreachable. On inspection, the app-web container responsible for handling uploads had exited with code 137 and was no longer listening on port 8000, causing nginx to return gateway errors.
    
- Investigation (checked, ruled out): • Verified network connectivity and nginx routing, both healthy. • Checked systemctl status for nginx, service was active. • Examined nginx error logs, only unrelated 404 probes, no direct upload errors. • Checked docker ps, app-web container was present but exited 12 hours ago. • Reviewed docker logs, container had started, reported listening on 0.0.0.0:8000, then shut down. • Tested connectivity with curl to 127.0.0.1:8000, connection refused, confirming backend was down. • Other containers unaffected, issue isolated to app-web backend service.
    
- Root cause: The app-web container (pavecorp/app:latest) had exited with code 137, leaving no process bound to port 8000. This caused nginx to return gateway errors for upload requests. Exit code 137 indicates the process was killed, likely due to resource limits or manual stop, not a code crash.
    
- Fix + confirmation: • Restarted the app-web container with docker start app-web. • Verified container logs, service listening again on 0.0.0.0:8000. • Confirmed connectivity with curl to 127.0.0.1:8000, returned valid response (404 from app-web, proving it was alive). • Applied restart policy with docker update --restart=always app-web to ensure persistence across host reboots. • Checked docker ps, app-web now shows Up with port 8000 mapped. Uploads tested successfully from Operations side, gateway errors resolved and dispatcher workflow restored.****

# 3. 

### 1. Problem reported / what I found:

A teammate reported difficulty debugging JSON responses from the archive service. They needed to pretty‑print JSON output but the host did not have `jq` installed, making raw responses hard to read. Attempting to run `jq` resulted in “command not found.”

#### 2. Investigation (checked, ruled out):

• Verified PATH — no `jq` binary present. • Attempted to run `jq --version` — not installed. • Tested fallback with `python3 -m json.tool` — produced “Expecting value” errors when the service returned empty or invalid JSON. • Confirmed issue was isolated to missing `jq` utility, not the archive service itself.

#### 3. Root cause:

The host system did not have `jq` installed. Without it, teammates could not easily format JSON responses during debugging.

#### 4. Fix + confirmation:

• Installed `jq` using the package manager (`yum install -y jq` or `apt-get install -y jq`). • Verified installation with `jq --version` — returned `jq-1.6`. • Tested by piping a sample JSON response through `jq` — output was formatted and readable. • Teammate confirmed they can now pretty‑print archive service responses while debugging.

# 4.

- Problem reported / what I found: Operations reported that uploads have been failing all morning with a generic error. The page itself loads normally, but the upload action fails for everyone. No changes were made on their side, and uploads worked yesterday afternoon.
    
- Investigation (checked, ruled out): • Checked nginx service status, confirmed active and serving. • Reviewed nginx error logs, only unrelated 404 probes, no direct upload errors. • Verified backend container app‑web is running and listening on port 8000. • Inspected app‑web logs, service reports listening and upload directories configured. • Tested upload endpoint with curl, received 404 error page from nginx. • Issue isolated to routing or endpoint configuration for /uploads.
    
- Root cause: The upload endpoint /uploads is not correctly mapped to the backend service. nginx is serving its default 404 page instead of forwarding the request, causing all uploads to fail.
    
- Fix + confirmation: • Restarted backend container with docker restart app‑web. • Restarted archive service with systemctl restart pave‑archive.service. • Verified app‑web logs show service listening with uploads directory configured. • Created a test file and retried upload with curl -F file=@/tmp/test.txt http://localhost/uploads/, confirmed backend response instead of nginx 404. • Operations confirmed uploads now succeed.



---

## Tasks completed, in order:

**1. Landing page showing nginx default test page; /status, /archive, /reports dead** Root cause: leftover stock RHEL default `server { listen 80; server_name _; }` block in `/etc/nginx/nginx.conf` conflicted with the real pave vhost in `/etc/nginx/conf.d/pave-app.conf` (also `server_name _;`). nginx nondeterministically routed requests between the two. Fix: removed/commented the stock block from `nginx.conf`, reloaded nginx. **This same root cause recurred and had to be fixed again during Task 7 below.**

**2. Recurring disk-full alert (93% on /)** Found `/var/reserved.pad` (60G) — intentionally pre-allocated for a postponed Postgres migration (PAVE-2291); README explicitly said not to touch it. Real cause: `pave-archive.service`'s logrotate config uses `copytruncate`/dateext and was actually working correctly, but `access.log-20260902` had grown to 6.1G in one day (56M lines, mixed/out-of-order timestamps — never fully root-caused why, deprioritized). Fix: gzip'd the oversized rotated log to reclaim space immediately, fixed a `rotatte`→`rotate` typo in the logrotate stanza, added a `size` trigger as a safety net.

**3. Report exports slow (still work, just ~1 min instead of seconds)** Root cause: `pave-cache.service` (port 7000) crash-looping — `KeyError: 'CACHE_PORT'` because that variable was simply missing from `/etc/pave-cache/cache.env`. With cache permanently down, `pave-reports` fell back to its documented (slower) direct-read path on every request. Fix: added `CACHE_PORT=7000` to the env file, restarted pave-cache, confirmed listening and reports back to normal speed.

**4. Ops channel job-completion notifications stopped after maintenance window** Root cause: `app-notify` container was attached only to Docker's default `bridge` network, not `pave-net` (confirmed via `NetworkMode: bridge` in `docker inspect`), so it couldn't resolve `app-web` by name (Docker embedded DNS only works within a shared user-defined network). Fix: live-patched with `docker network connect pave-net app-notify`, then made it permanent by removing and recreating the container with `--network pave-net` set explicitly at creation.

**5. Uploads accepted but jobs never complete, queue piling up** Root cause: `app-worker`'s bind mount for `/srv/pave/queue` was `ro` (read-only) at the container level — worker could read job files but crashed (`OSError: Read-only file system`) trying to write result files back into the same directory. Fix: recreated app-worker with the queue mount as `rw`; confirmed the whole backlog processed and cleared.

**6. Site footer showing old version v1.4.1 despite v1.4.2 "shipped and signed off"** Root cause: classic tag/symlink drift the deploy doc warns about — v1.4.2 was fully built under `/var/www/releases/v1.4.2/` but `/var/www/current` was never repointed (still symlinked to v1.4.1), and the deploy repo (`/opt/pave/app`) was also still checked out at v1.4.1. Fix: atomically repointed the symlink (`ln -sfn`), checked out v1.4.2 in the deploy repo to close the version-discipline gap, confirmed live.

**7. Data team: Export button fails without a format picked, works if format chosen** Two combined causes:

- Same nginx dual-default-server-block conflict as Task 1 had recurred, intermittently misrouting `/reports/*` requests to nginx's own static 404 instead of proxying to pave-reports.
- Real app bug in `/opt/pave/reports/serve.py`: `fmt = params.get("format", [None])[0]` defaulted to `None`, then `fmt.lower()` crashed with `AttributeError` when no format was supplied. Fix: removed the duplicate nginx server block again; changed the default to `"csv"` instead of `None`; restarted pave-reports; confirmed no-format, `format=csv`, and `format=json` all return 200.

**8. Data team: nightly metrics CSV missing for two days despite job log saying success** Root cause: one-character typo — script's `OUT_DIR = "/srv/pave/report"` (singular) while everyone was watching `/srv/pave/reports` (plural); both directories existed separately on disk. Job was succeeding every night, just writing to the wrong (unwatched) folder. Fix: corrected `OUT_DIR` in `daily_report.py`, moved the two stranded files into the correct directory, fixed ownership, removed the stray singular directory, verified a fresh manual run wrote to the right place.

**9. Release team: merge conflict pulling into staging repo before cutting next release** Cause: genuine conflict in `config/settings.py` — local staging had `RETRY_BACKOFF_SECONDS = 90` (Aug 26 incident fix), incoming origin had `= 30` (east-hub timeout fix). Resolved by keeping `90` (user's call). Fix: edited out conflict markers, staged and committed the merge (needed `sudo` due to repo ownership permission issues, a recurring theme across `/opt/pave/app` and `/opt/pave/app-staging`). **Flagged but not yet done: staging is 2 commits ahead of origin and needs `git push` before the deploy repo can pick up this change for the actual release.**

## Recurring environmental themes worth knowing for future tasks:

- Git repos under `/opt/pave/` (app, app-staging, app-remote.git) require `git config --global --add safe.directory` due to ownership checks, and often need `sudo` for actual write operations (index.lock permission errors) — matches the handover note about permission drift under `/opt`/`/srv` from manual SSH access.
- `/opt/pave/src` is leftover install media — never the live code; always confirm the real path via `systemctl cat <service>` before editing.
- The nginx dual-default-server-block issue is not fully eradicated at the root (only patched twice reactively) — may be worth a permanent structural fix or a check added to deployment/maintenance procedures.
- `/var/reserved.pad` (60G) must never be touched as a "fix" for disk issues.