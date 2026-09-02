Here’s the **full structured summary of all 13 tasks**, with the actual input for each section (Problem, Investigation, Root Cause, Fix + Confirmation).

### **Task 1 — Status Page Blank After Reboot**

- **Reported by:** Ops leads
    
- **Context:** Dispatch room wall-mounted display shows platform health.
    
- **Sensing/Issue:** “The status board in dispatch is blank again after reboot.”
    

**1. Problem reported / what I found:** The dispatch room’s wall-mounted status display was blank after reboot. The `pave-statuspage.service` was not starting automatically and showed as inactive (dead).

**2. Investigation (checked, ruled out):**

- Verified network connectivity and nginx routing, both healthy.
    
- Checked `systemctl status pave-statuspage.service`, showed inactive immediately after boot.
    
- Examined `journalctl -u pave-statuspage.service`, logs indicated missing Python path environment variable.
    
- Confirmed service file lacked `EnvironmentFile=/etc/pave-cache/cache.env` and `After=network.target`.
    
- Docker containers unaffected.
    

**3. Root cause:** Service unit missing environment configuration and dependency ordering.

**4. Fix + confirmation:** Added `EnvironmentFile=/etc/pave-cache/cache.env` and `After=network.target`; reloaded systemd; enabled service; reboot confirmed auto-start.

### **Task 2 — Uploads Failing (Gateway Error)**

- **Reported by:** Operations
    
- **Context:** Uploads of carrier paperwork.
    
- **Sensing/Issue:** “Nobody can upload anything, page shows gateway error.”
    

**1. Problem reported / what I found:** Uploads failing; nginx healthy but app-web container exited with code 137.

**2. Investigation:**

- nginx active.
    
- `docker ps` → app-web exited.
    
- Logs showed container started then killed.
    
- Curl to 127.0.0.1:8000 → refused.
    

**3. Root cause:** app-web container killed (exit 137).

**4. Fix + confirmation:** Restarted container; applied restart policy (`docker update --restart=always`); verified uploads restored.

### **Task 3 — Missing** `jq` **Utility**

- **Reported by:** Teammate
    
- **Context:** Debugging JSON responses.
    
- **Sensing/Issue:** “jq isn’t installed, need pretty-print.”
    

**1. Problem reported / what I found:** No `jq` installed; raw JSON unreadable.

**2. Investigation:**

- PATH check → no binary.
    
- `jq --version` → not found.
    
- Python fallback failed.
    

**3. Root cause:** Utility absent.

**4. Fix + confirmation:** Installed `jq`; verified with sample JSON; teammate confirmed usability.

### **Task 4 — Uploads Failing (404 Error)**

- **Reported by:** Operations
    
- **Context:** Upload action fails, page loads fine.
    
- **Sensing/Issue:** “Uploads failing all morning with generic error.”
    

**1. Problem reported / what I found:** Uploads failing with 404; app-web running.

**2. Investigation:**

- nginx active.
    
- app-web listening.
    
- Curl to `/uploads` → nginx 404.
    

**3. Root cause:** nginx routing misconfigured for `/uploads`.

**4. Fix + confirmation:** Restarted app-web + pave-archive; curl test confirmed backend response; Ops confirmed success.

### **Task 5 — Landing Page Conflict**

- **Reported by:** Marketing
    
- **Context:** Customer-facing landing pages.
    
- **Sensing/Issue:** “Landing page shows HTTP Server Test Page; links dead.”
    

**1. Problem reported / what I found:** Landing page misrouted; `/status`, `/archive`, `/reports` dead.

**2. Investigation:** Duplicate default `server_name _;` block in nginx configs.

**3. Root cause:** Conflict between stock nginx block and pave vhost.

**4. Fix + confirmation:** Removed duplicate block; reloaded nginx. _(⚠ Recurring issue, not permanently fixed.)_

### **Task 6 — Disk-Full Alert**

- **Reported by:** Monitoring / Marcus
    
- **Context:** 93% disk utilization recurring.
    
- **Sensing/Issue:** “Space cleared before, problem came back.”
    

**1. Problem reported / what I found:** Disk nearly full; `/var/reserved.pad` untouched.

**2. Investigation:** Oversized rotated log (`access.log-20260902` grew to 6.1G).

**3. Root cause:** Logrotate worked but log exploded unexpectedly.

**4. Fix + confirmation:** Gzip’d log; fixed typo in logrotate; added size trigger. _(⚠ Underlying cause not root-caused.)_

### **Task 7 — Slow Report Exports**

- **Reported by:** Ops + Data team
    
- **Context:** Reports slower than usual.
    
- **Sensing/Issue:** “Exports take a minute instead of seconds.”
    

**1. Problem reported / what I found:** Reports slow; cache service crash-looping.

**2. Investigation:**

- pave-cache logs → `KeyError: 'CACHE_PORT'`.
    
- Port 7000 not listening.
    

**3. Root cause:** Missing `CACHE_PORT` in env file.

**4. Fix + confirmation:** Added variable; restarted service; confirmed speed restored.

### **Task 8 — Ops Notifications Broken**

- **Reported by:** Operations
    
- **Context:** Job completion messages missing.
    
- **Sensing/Issue:** “Stopped getting ‘file ready’ messages.”
    

**1. Problem reported / what I found:** Files processed but no notifications.

**2. Investigation:**

- app-notify logs → DNS failures.
    
- Network check → app-notify on default bridge, not `pave-net`.
    

**3. Root cause:** Container misconfigured network.

**4. Fix + confirmation:** Reattached to `pave-net`; recreated container with correct network; confirmed notifications restored.

### **Task 9 — Jobs Stuck in Queue**

- **Reported by:** Operations
    
- **Context:** Uploads accepted, jobs never complete.
    
- **Sensing/Issue:** “Jobs are stacking up and never finishing.”
    

**1. Problem reported / what I found:** Uploads accepted but jobs never completed; queue backlog growing.

**2. Investigation:**

- app-worker logs → `OSError: Read-only file system`.
    
- Queue ownership correct.
    
- Host write test succeeded.
    
- Container mount inspection → queue bind-mounted as read-only.
    

**3. Root cause:** app-worker queue mount configured read-only.

**4. Fix + confirmation:** Recreated container with rw mount; backlog cleared.

### **Task 10 — Version Drift**

- **Reported by:** Marketing
    
- **Context:** Release v1.4.2 shipped, site shows v1.4.1.
    
- **Sensing/Issue:** “Site footer still says v1.4.1.”
    

**1. Problem reported / what I found:** Site serving old version despite new release built.

**2. Investigation:**

- `/var/www/current` symlink still pointed to v1.4.1.
    
- Deploy repo checked out at v1.4.1.
    

**3. Root cause:** Symlink/tag drift; deploy process incomplete.

**4. Fix + confirmation:** Repointed symlink to v1.4.2; checked out v1.4.2 in deploy repo; confirmed live.

### **Task 11 — Export Button Crash**

- **Reported by:** Data team
    
- **Context:** Reports export control.
    
- **Sensing/Issue:** “Export fails unless format chosen.”
    

**1. Problem reported / what I found:** Export button failed without format; worked with format.

**2. Investigation:**

- nginx misrouting due to duplicate block.
    
- App traceback: `NoneType.lower()` crash.
    

**3. Root cause:** nginx misrouting + app bug (missing format default).

**4. Fix + confirmation:** Removed duplicate nginx block; defaulted format to `"csv"`; restarted service; confirmed all formats work.

### **Task 12 — Missing Metrics CSV**

- **Reported by:** Data team
    
- **Context:** Nightly job writes metrics CSV.
    
- **Sensing/Issue:** “Metrics file missing for two days.”
    

**1. Problem reported / what I found:** Metrics CSV missing in `/srv/pave/reports`.

**2. Investigation:**

- Cron fired correctly.
    
- Job log showed success.
    
- Found output path typo: `/srv/pave/report` vs `/srv/pave/reports`.
    

**3. Root cause:** Typo in output directory.

**4. Fix + confirmation:** Corrected path; moved stranded files; removed stray directory; confirmed metrics present.

### **Task 13 — Merge Conflict in Staging Repo**

- **Reported by:** Release team
    
- **Context:** Next release staged before deploy.
    
- **Sensing/Issue:** “Pull into staging failed with merge conflict.”
    

**1. Problem reported / what I found:** Merge conflict in `config/settings.py`.

**2. Investigation:**

- Conflict isolated to one line (`RETRY_BACKOFF_SECONDS`).
    
- Staging had 90; origin had 30.
    

**3. Root cause:** Independent changes caused genuine conflict.

**4. Fix