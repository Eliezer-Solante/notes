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