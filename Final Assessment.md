ssh esolante@bastion-1.acad.opswerks.net

Acad_student-esolante

ssh esolante@esolante-a1t-inf-compre1.acad.opswerks.net

<p>1. Problem reported / what I found:</p> <p>The dispatch room’s wall-mounted status display was blank after the maintenance reboot. The <code>pave-statuspage.service</code> (port 8081) was not running, and the Ops leads confirmed it had failed to start automatically after each reboot.</p>

<p>2. Investigation (checked, ruled out):</p> <p>• Verified network connectivity and nginx routing — both healthy.<br> • Checked <code>systemctl status pave-statuspage.service</code> — showed “inactive (dead)” immediately after boot.<br> • Examined <code>journalctl -u pave-statuspage.service</code> — logs indicated missing Python path environment variable.<br> • Confirmed the service file under <code>/etc/systemd/system/</code> lacked <code>EnvironmentFile=/etc/pave-cache/env</code> or equivalent.<br> • Docker containers unaffected; issue isolated to host service startup.</p>

<p>3. Root cause:</p> <p>The <code>pave-statuspage.service</code> unit did not include the required environment configuration, so systemd attempted to start it before its dependencies were ready. It succeeded only when manually started later, after the environment was loaded. The missing <code>After=network.target</code> and absent <code>EnvironmentFile</code> entries prevented reliable boot‑time startup.</p>

<p>4. Fix + confirmation:</p> <p>• Edited <code>/etc/systemd/system/pave-statuspage.service</code> to include:<br> <code>After=network.target</code><br> <code>EnvironmentFile=/etc/pave-cache/env</code><br> • Ran <code>systemctl daemon-reload</code> and <code>systemctl enable pave-statuspage.service</code>.<br> • Rebooted host to confirm persistence — service auto‑started, port 8081 reachable, and display showed green status indicators.<br> • Verified via <code>journalctl -u pave-statuspage.service</code> that startup completed cleanly.<br> ✅ Issue resolved; status board remains active after reboot.</p>