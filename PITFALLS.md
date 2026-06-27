# Pitfall Registry

> Proven through real failures on real hardware. Each entry tagged: **BUG** (active footgun), **FIX** (resolved with workaround), **DESIGN** (architectural decision).

---

### BUG — SSH + HOME sandboxing

Profiles sandbox `$HOME` to `~/.hermes/profiles/<name>/home`. SSH identity paths **must** be absolute:

```bash
# WRONG — sandboxed HOME doesn't have .ssh/
ssh user@<SIFT-VM-IP> "command"

# RIGHT
ssh -i /home/user/.ssh/id_rsa user@<SIFT-VM-IP> "command"
```

The `sift-exec.sh` wrapper handles this automatically. Every new SSH wrapper must use absolute key paths.

---

### BUG — ConnectTimeout too short

Use **10 seconds** for SSH to SIFT VM. 3 seconds causes intermittent failures when run immediately after Docker commands (system briefly busy from container teardown).

---

### FIX — SIFT VM cold boot race

`hermes-sift-vm.service` has an `ExecStartPre` that polls vmnet8 for `<SIFT-VM-SUBNET>` (up to 30s) before launching the VM. This eliminates the VMware networking race where the VM boots before the virtual network is ready.

Manual recovery if the service fails:
```bash
vmrun -T ws start /home/user/vmware/SIFT/SIFT.vmx nogui
# Wait 25–45 seconds for SSH to come up
```

---

### FIX — SIFT VM MUST use NAT, not bridged, on WiFi

Bridged networking over WiFi is fundamentally unreliable:
- DHCP leases expire between sessions
- APs drop VM MAC addresses
- VM loses all IPv4 connectivity

**Fix (definitive):**
```bash
# In VMX file:
ethernet0.connectionType = "nat"

# In /etc/vmware/vmnet8/nat/nat.conf:
[incomingtcp]
2222 = <SIFT-VM-IP>:22

# Restart VMware networking:
sudo vmware-networks --stop && sudo vmware-networks --start
```

Then target `localhost:2222` — IP never changes.

---

### FIX — Foreground dispatch timeout (600s)

`hermes -z` in foreground mode has a **600-second hard limit**. Full pentest engagements take 8–12 minutes and WILL time out before report generation. The dispatch is killed, no PDF is produced, and the error is opaque.

**Fix:** Always use background dispatch for full engagements:
```bash
# In terminal tool: background=true, notify_on_complete=true
hermes -z "Run full pentest engagement..." --profile pentest
# Then: process(action='wait', session_id='...')
```

---

### BUG — WireGuard VPN detection

WireGuard interfaces report `operstate` as `"unknown"` in `/sys/class/net/wg0/operstate`, NOT `"up"`. Checking `== "up"` always fails.

**Correct check:**
```bash
[ "$(cat /sys/class/net/wg0/operstate)" != "down" ]
```
Any non-down state is active.

---

### BUG — Pentest profile .env required

The pentest profile needs its own `.env` with provider API keys. Without it:
```
ERROR: Provider '&lt;your-provider&gt;' has no API key.
```

**Fix:**
```bash
cp ~/.hermes/.env ~/.hermes/profiles/pentest/.env
chmod 600 ~/.hermes/profiles/pentest/.env
```

---

### BUG — Docker entrypoint gotchas

- `forensics-volatility3:2.7.0` — entrypoint **IS** the volatility3 binary. Commands start with flags directly: `-f dump.mem windows.pslist` (NOT `vol -f dump.mem ...`)
- `forensics-mft-tools:1.2.0.0` — entrypoint is bash. Use `--entrypoint python3` to run Python tools.

---

### BUG — `command -v`, not `which`

Use `command -v` in canary scripts — works for both PATH tools and absolute paths. `which` is unreliable in scripts.

RegRipper is at `/usr/lib/regripper/rip.pl` (not on PATH). Check with full path, not `command -v`.

---

### FIX — 7zip for AES-encrypted ZIPs

Standard `unzip` fails on compression method 99 (AES encryption):
```
unsupported compression method 99
```

**Fix:**
```bash
sudo apt install p7zip-full   # Ubuntu ≤ 24.04
# or
sudo apt install 7zip         # Ubuntu ≥ 26.04

# Binary is '7z', not '7zz'
7z x -pPASSWORD -y file.zip
```

---

### FIX — MemProcFS mount resilience

Large memory dumps (>4GB) take 60–120 seconds to initialize. Stale FUSE mounts from a killed process leave a broken mount point:
```
Transport endpoint is not connected
```

**Fix:**
```bash
fusermount -u /home/user/forensics/mounts/mem
rm -rf /home/user/forensics/mounts/mem
mkdir -p /home/user/forensics/mounts/mem
# Then re-mount
```

Mount point must be user-writable (`/home/user/forensics/mounts/`), not `/mnt`.

---

### DESIGN — Script coding patterns

Proven through review of all automation scripts:

1. **No `set -e` in bring-up/shutdown scripts.** These scripts report degradation, never abort. A failing canary or unreachable VM should not kill the script. Use `set -uo pipefail` only.

2. **stderr/stdout split for `$(...)` capture.** Scripts that return a value (like `CASE_ID`) print banner/metadata to stderr (`>&2`) and ONLY the value to stdout.

3. **`sudo -n` for non-interactive automation.** If passwordless sudo isn't configured, the script fails fast with a clear message instead of hanging on a password prompt.

4. **Glob-based cleanup, not hardcoded paths.** Instead of `/mnt/mem`, glob `/home/user/forensics/mounts/*/` to catch any mount point.

---

### DESIGN — Ponytail minimalism

Forensics lab: **3 core scripts + 1 data file.** Everything else (case creation, evidence registration, IOC scanning, report generation) is handled by the forensics profile's skills through agent reasoning. Scripts are only for the deterministic, repetitive, footgun-prone bits:

| File | Lines | Why it exists |
|------|-------|---------------|
| `session-canary.sh` | 126 | Validates all tools — deterministic, the agent can't validate itself |
| `sift-exec.sh` | 24 | SSH wrapper — prevents HOME-sandbox key-path footgun |
| `down.sh` | 40 | Cleanup — prevents stale mounts, running VMs, FUSE leftovers |
| `tool-catalog.yaml` | 120 | Data — single source of truth for versions, entrypoints, pitfalls |
