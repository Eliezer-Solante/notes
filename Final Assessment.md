ssh esolante@bastion-1.acad.opswerks.net

Acad_student-esolante

ssh esolante@esolante-a1t-inf-compre1.acad.opswerks.net


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

