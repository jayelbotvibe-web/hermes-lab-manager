# Replicating This Setup

> 10 steps from bare metal Ubuntu to a working multi-profile Hermes lab with forensics and pentest dispatch.

---

## Prerequisites

- **Ubuntu 24.04+** (tested on 26.04, kernel 7.0)
- **30 GB+ free disk, 8 GB+ RAM** (the reference machine: i7-11800H, 30 GB RAM, 468 GB NVMe)
- **VMware Workstation 17.x** (for SIFT VM)
- **Docker + Docker Compose v2**
- **[Hermes Agent v0.17.0+](https://github.com/NousResearch/hermes-agent)**

```bash
sudo apt install -y docker.io docker-compose-v2 wireguard-tools cryptsetup pandoc weasyprint gh
```

---

## Step 1: Clone Repos

```bash
git clone https://github.com/jayelbotvibe-web/hermes-forensics-lab.git ~/hermes-forensics-lab
git clone https://github.com/jayelbotvibe-web/hermes-pentest-lab.git ~/pentest-repo
```

---

## Step 2: Create LUKS Vaults

```bash
# Forensics vault (30 GB)
dd if=/dev/zero of=/home/$USER/forensics.img bs=1M count=30720
sudo cryptsetup luksFormat /home/$USER/forensics.img
sudo cryptsetup luksOpen /home/$USER/forensics.img forensics_crypt
sudo mkfs.ext4 /dev/mapper/forensics_crypt
sudo mkdir -p /home/$USER/forensics
sudo mount /dev/mapper/forensics_crypt /home/$USER/forensics
sudo chown $USER:$USER /home/$USER/forensics

# Pentest vault (10 GB)
dd if=/dev/zero of=/home/$USER/pentest-vault.img bs=1M count=10240
sudo cryptsetup luksFormat /home/$USER/pentest-vault.img
sudo cryptsetup luksOpen /home/$USER/pentest-vault.img pentest-vault
sudo mkfs.ext4 /dev/mapper/pentest-vault
sudo mkdir -p /home/$USER/pentest
sudo mount /dev/mapper/pentest-vault /home/$USER/pentest
sudo chown $USER:$USER /home/$USER/pentest
```

---

## Step 3: Set Up Keyfiles (Passwordless Mount)

```bash
echo -n '<your-password>' > /home/$USER/.forensics-keyfile
echo -n '<your-password>' > /home/$USER/.pentest-keyfile
chmod 600 /home/$USER/.forensics-keyfile /home/$USER/.pentest-keyfile

# Register forensics LUKS slot with keyfile
sudo cryptsetup luksAddKey /home/$USER/forensics.img /home/$USER/.forensics-keyfile
sudo cryptsetup luksAddKey /home/$USER/pentest-vault.img /home/$USER/.pentest-keyfile

# Add forensics to crypttab (noauto — manual mount only)
echo "forensics_crypt /home/$USER/forensics.img /home/$USER/.forensics-keyfile luks,noauto" | \
  sudo tee -a /etc/crypttab
```

---

## Step 4: Set Up SIFT VM

1. Download SIFT Workstation OVA from [SANS Portal](https://www.sans.org/tools/sift-workstation/)
2. Import into VMware Workstation
3. Configure **NAT networking (vmnet8)** — NOT bridged:
   - In VM settings: Network Adapter → NAT
   - Power on VM, set static IP: `172.16.146.128`
4. Add port forward (optional, for SSH via localhost):
   ```bash
   # /etc/vmware/vmnet8/nat/nat.conf
   [incomingtcp]
   2222 = 172.16.146.128:22
   sudo vmware-networks --stop && sudo vmware-networks --start
   ```
5. Copy SSH key:
   ```bash
   ssh-copy-id sansforensics@172.16.146.128
   ```

---

## Step 5: Build Docker Images

```bash
# Forensics tools
docker build -t forensics-volatility3:2.7.0 ~/hermes-forensics-lab/tools/volatility/
docker build -t forensics-plaso:20240512 ~/hermes-forensics-lab/tools/plaso/
docker build -t forensics-mft-tools:1.2.0.0 ~/hermes-forensics-lab/tools/mft-tools/

# Pentest tools
cd ~/pentest-repo/docker && docker compose build
```

---

## Step 6: Copy Forensics Scripts into Vault

The `hermes-forensics-lab` repo contains the scripts — copy them into the LUKS mount:

```bash
cp ~/hermes-forensics-lab/scripts/*.sh /home/$USER/forensics/scripts/
cp ~/hermes-forensics-lab/scripts/*.py /home/$USER/forensics/scripts/
cp ~/hermes-forensics-lab/reports/templates/*.html /home/$USER/forensics/templates/
cp ~/hermes-forensics-lab/tools/tool-catalog.yaml /home/$USER/forensics/tools/
```

Then update paths in scripts to use vault paths (e.g., template path → `/home/niel/forensics/templates/`).

---

## Step 7: Create Hermes Profiles

```bash
hermes profile create forensics
hermes profile create pentest
```

Configure each profile's `config.yaml`:
```yaml
# ~/.hermes/profiles/forensics/config.yaml
agent:
  max_turns: 120
  tool_use_enforcement: true
terminal:
  backend: local          # MUST be local — needs host file access
  timeout: 600
approvals:
  mode: manual            # Never YOLO on evidence
```

Copy API keys:
```bash
cp ~/.hermes/.env ~/.hermes/profiles/forensics/.env
cp ~/.hermes/.env ~/.hermes/profiles/pentest/.env
chmod 600 ~/.hermes/profiles/{forensics,pentest}/.env
```

---

## Step 8: Install Lab Manager Skill

```bash
mkdir -p ~/.hermes/skills/devops/lab-manager
# Copy SKILL.md from this repo's skill/ directory
# Or create from the template below
```

The skill enforces the routing rule: default profile never runs lab tools. Copy `skill/SKILL.md` to `~/.hermes/skills/devops/lab-manager/SKILL.md`.

---

## Step 9: Install Health Check

```bash
cat > ~/.hermes/scripts/check-labs << 'EOF'
#!/bin/bash
set -e
echo "=== PENTEST ==="
bash /home/$USER/pentest-repo/scripts/pentest-canary.sh 2>&1 | tail -3
echo ""
echo "=== FORENSICS ==="
bash /home/$USER/forensics/scripts/session-canary.sh 2>&1 | tail -3
EOF
chmod +x ~/.hermes/scripts/check-labs
```

---

## Step 10: Verify

```bash
# Check both labs
bash ~/.hermes/scripts/check-labs

# Test dispatch to each profile
hermes -z "Run canary and report pass/fail." --profile forensics
hermes -z "Run canary and report pass/fail." --profile pentest

# Full capability drill
hermes -z "Run canary, verify all Docker images, verify MemProcFS, verify SIFT tools" --profile forensics
```

Expected output:
```
=== PENTEST ===
=== Results: 14 passed, 0 failed ===
✓ All services operational

=== FORENSICS ===
=== Results: 18 passed, 0 failed ===
✓ All runtimes operational
```

---

## Post-Setup: Daily Workflow

```bash
# 1. Check labs
bash ~/.hermes/scripts/check-labs

# 2. Dispatch forensic task
hermes -z "Run canary. Analyze /home/$USER/Downloads/capture.pcap.
  Use tshark on SIFT VM. Open case, register evidence, identify IOCs.
  Generate data-first report with screenshots." --profile forensics

# 3. Dispatch pentest task
hermes -z "Run canary. Passive OSINT on example.com. Zero packets.
  Return subdomains, emails, exposed tech." --profile pentest
```

---

## Troubleshooting

| Symptom | Check |
|---------|-------|
| "Provider has no API key" | `.env` missing in profile directory |
| SSH connection refused to SIFT | VM not running — `vmrun list` |
| Docker "Cannot connect" | `sudo systemctl start docker` |
| "Transport endpoint not connected" | Stale FUSE mount — `fusermount -u` |
| `hermes -z` times out at 600s | Use background mode for full engagements |
| Forensics vault won't mount | Keyfile wrong — verify with `sudo cryptsetup luksOpen --test-passphrase` |
