---
name: lab-manager
description: "Route tasks to forensics/pentest profiles via one-shot dispatch. Health checks, folder layout, pitfalls, recovery."
version: 1.4.0
category: devops
---

# Lab Manager — ponytail edition

Route forensics and pentest tasks to the right profile. Stay thin — no tool execution.

## HARD ROUTING RULE (read first — no exceptions)

When the user asks for pentesting, forensics, or any lab work, you **MUST** dispatch via:

```bash
hermes -z "<self-contained prompt>" --profile <pentest|forensics>
```

**You do NOT:**
- Load domain skills manually (`pentest-engage`, `pentest-ops`, etc.)
- Run tools directly (nmap, volatility, ffuf, etc.)
- Execute scan commands yourself
- Start or check target services (Juice Shop, DVWA, etc.) — the worker profile handles its own targets
- Try to do lab work from the default profile

**You ONLY route.** The worker profile has its own skills, SOUL, memory, model config, and tool environment. Your job is to construct the prompt and call `hermes -z --profile`. The worker profile handles execution, gates, and report generation.

If you catch yourself about to run a pentest or forensics tool, STOP. You're in the wrong profile. Dispatch.

### Vault enforcement (structural — not memory-based)

If dispatch fails (timeout, profile unreachable), **report the failure to the user**. Do NOT fall back to direct execution. The LUKS vault makes this structural: forensics templates, scripts, and tooling live under the LUKS mount. When the vault is locked, these files literally do not exist on the filesystem and any attempt to access them gets ENOENT. This is the enforcement mechanism — the default profile cannot read what isn't there.

Dispatch timeout -> tell user "forensics profile unreachable after 120s" and stop. Never bypass.

## Dispatch Verification Flow

```bash
# 1. Verify CLI one-shot mode exists
hermes --help | grep '\-z'

# 2. Test basic dispatch to each profile
hermes -z "Say the hostname and exit." --profile forensics
hermes -z "Say the hostname and exit." --profile pentest

# 3. Test canary dispatch (full tool validation)
hermes -z "Run the session canary and report pass/fail." --profile forensics
hermes -z "Run the pentest canary and report pass/fail." --profile pentest

# 4. Full capability drill
hermes -z "Run canary, verify all Docker images, verify MemProcFS, verify SIFT tools" --profile forensics
hermes -z "Run canary, verify containers, verify VPN, run passive OSINT on a safe domain" --profile pentest
```

`-z` prints ONLY the final response to stdout. No banner, no spinner, no tool previews.

## Silent Dispatch Pattern

When the user asks for lab work from the default profile, construct the prompt and dispatch immediately. Do not ask permission. Do not explain. Just route:

```bash
# Pentest
hermes -z "Run full pentest engagement against example.com. Scope: example.com and *.example.com. Run canary first. Execute all 7 workflow phases including report generation. Report the PDF path when done." --profile pentest

# Forensics
hermes -z "Run full memory forensics analysis on /home/USER/cases/CASE/memory.dmp. Run canary first. Produce findings. Report summary when done." --profile forensics
```

**Timeout pitfall:** Foreground `hermes -z` has a 600s hard limit. Full engagements take 8-12 minutes and WILL time out. Always use background dispatch for full engagements: `terminal(background=true, notify_on_complete=true)`. Canary-only checks (~60s) are fine in foreground.

**Prompt construction checklist:**
- Include target/IP/path (absolute — worker profiles sandbox $HOME)
- Include scope boundaries
- Instruct canary first ("Run canary first, report degraded tools")
- Request specific output (report path, findings summary)
- Keep it brief — the worker profile's skills fill in the workflow

## Health

```bash
bash /home/niel/.hermes/scripts/check-labs
```

## Folder Structure

See `references/folder-structure.md` for the canonical layout of both labs.

Quick reference:
- **Forensics**: cases live in `cases/INC-YYYY-MMDD-NNNN/`, reports per-case, templates in vault
- **Pentest**: engagements in `engagements/<name>/`, scripts/skills in `pentest-repo/`, not in vault
- **Pentest vault**: only encrypted-at-rest data (engagements, keys, wordlists, tools, docker volumes)

## Key Pitfalls

### SSH + HOME sandboxing
Profiles sandbox HOME. SSH MUST use absolute identity: `-i /home/niel/.ssh/id_rsa`.
Never `~/.ssh/` — the tilde resolves to `~/.hermes/profiles/<name>/home`.

### ConnectTimeout
Use 10 seconds for SSH to SIFT VM (172.16.146.128 vmnet8 NAT).
3 seconds causes intermittent failures after Docker commands.

### SIFT VM cold boot
`hermes-sift-vm.service` has `ExecStartPre` that polls vmnet8 for 172.16.146.x before launching VM. Manual recovery: `vmrun -T ws start /home/niel/vmware/SIFT/SIFT.vmx nogui` (wait 25-45s for SSH).

### SIFT VM MUST use NAT, not bridged, on WiFi
Bridged networking over WiFi causes DHCP lease expiration and IP drift. Fix: NAT (vmnet8) with static IP.

### Forensics crypttab
Uses LUKS with `noauto` — manual mount only via `forensics-up.sh`. Vault auto-mounts without `noauto`.

### Pentest vault persistence
LUKS does NOT survive reboot. Manual mount + Docker container start needed.

### Pentest .env required
At `~/.hermes/profiles/pentest/.env`. Copied from default profile.
Without it: "Provider has no API key."

### Docker entrypoints
- `forensics-volatility3:2.7.0`: entrypoint IS volatility3 binary — no `vol` prefix
- `forensics-mft-tools:1.2.0.0`: entrypoint is bash — use `--entrypoint python3`

### WireGuard VPN detection
WireGuard interfaces report `operstate` as "unknown" in `/sys/class/net/wg0/operstate`,
NOT "up". Checking `== "up"` always fails. Use `!= "down"`.

### command -v for tool checks
Use `command -v` not `which` in canary — works for both PATH tools and absolute paths.
RegRipper is at `/usr/lib/regripper/rip.pl` (not on PATH). Check with full path.

### hermes -z foreground timeout (600s hard limit)
Full engagements take 8-12 minutes. `hermes -z` foreground hits the 600-second timeout.
Fix: always use background dispatch with `notify_on_complete=true`.

`hermes -z --profile <name>` runs with the calling shell's cwd, NOT the profile's configured `terminal.cwd`. Both profiles use absolute paths everywhere — not a problem in practice, but explain in dispatch prompts if relative paths appear.

## Files That Matter

| File | Why |
|------|-----|
| `~/.hermes/scripts/check-labs` | One command, both canaries |
| `~/forensics/scripts/session-canary.sh` | 18 checks, 3 runtimes |
| `~/forensics/scripts/sift-exec.sh` | SSH footgun prevention |
| `~/forensics/scripts/down.sh` | Cleanup |
| `~/forensics/tools/tool-catalog.yaml` | Tool versions, pitfalls |
| `~/forensics/templates/` | Report templates (data-first + timeline) |
| `~/pentest-repo/scripts/pentest-canary.sh` | 14 checks |
| `~/pentest-repo/scripts/pentest-up.sh` | Container startup |
| `~/pentest-repo/scripts/pentest-down.sh` | Graceful shutdown |
