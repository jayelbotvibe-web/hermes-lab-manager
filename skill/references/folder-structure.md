# Canonical Folder Structure — June 2026

Post-reorganization. Ponytail-pruned. No duplication between repo and vault.

## Forensics (`/home/user/forensics/`)

```
forensics/
├── cases/
│   ├── INC-YYYY-MMDD-NNNN/    ← per-case isolation
│   │   ├── audit/             ← JSONL audit trail
│   │   ├── evidence/          ← original evidence (read-only)
│   │   ├── raw/               ← tool stdout/stderr
│   │   └── reports/           ← final reports
│   └── ...
├── fixtures/
│   ├── evidence-locker/       ← git repo of test evidence
│   └── expected_outputs/      ← diff targets for validation
├── logs/                      ← session and tool logs
├── mounts/mem/                ← MemProcFS FUSE mount point
├── scripts/
│   ├── session-canary.sh      ← 20 checks (12 tools + 8 env), 3 runtimes
│   ├── sift-exec.sh           ← SSH wrapper (key-path-safe)
│   └── down.sh                ← cleanup (LUKS, VM, FUSE)
├── tools/
│   ├── tool-catalog.yaml      ← single source of truth
│   ├── mft-tools/             ← Docker build context
│   ├── plaso/                 ← Docker build context
│   ├── registry/              ← Docker build context
│   └── volatility/            ← Docker build context
└── FORENSICS-BUILD-RUNBOOK.md ← build guide
```

Rules:
- Reports live in case directories, NOT in root `reports/`
- Evidence paths: ALWAYS absolute (`/home/user/forensics/...`)
- HOME is sandboxed in profiles → `~/forensics/` resolves wrong
- LUKS vault: `forensics.img` (30GB) with `noauto` crypttab using `.forensics-keyfile`

## Pentest Vault (`/home/user/pentest/` — LUKS encrypted)

```
pentest/
├── archives/                   ← encrypted engagement archives (age)
├── docker/
│   ├── docker-compose.yml
│   └── volumes/
│       └── neo4j/              ← BloodHound graph DB (was scans/neo4j-data/)
├── engagements/
│   └── <name>/                 ← per-engagement isolation
│       ├── audit/              ← JSONL audit trail
│       ├── evidence/           ← screenshots, client docs
│       ├── raw/                ← nmap XML, nuclei JSON
│       ├── reports/            ← generated PDF/MD
│       ├── ENGAGEMENT.yaml     ← metadata
│       ├── scope.json          ← scope definition
│       └── findings.db         ← SQLite findings database
├── keys/                       ← GPG + age encryption keys
├── reference/                  ← methodology docs, sample report guides
├── tools/                      ← burpsuite_community.jar
└── wordlists/                  ← SecLists + PayloadsAllTheThings
```

Rules:
- Vault = encrypted-at-rest data ONLY (engagements, keys, wordlists, tools, docker volumes)
- Scripts and skills live in `pentest-repo/`, NOT in vault (was duplicated, now deleted)
- `reports/` at root deleted — reports live in engagement dirs
- `scope/` at root deleted — scope lives per-engagement or as template in repo
- `scans/` deleted — neo4j is Docker volume data, not scan output
- `learning/` + `methodology/` merged into `reference/`
- `venv/` moved to `pentest-repo/venv/` (doesn't need LUKS encryption)

## Pentest Repo (`/home/user/pentest-repo/` — git versioned)

```
pentest-repo/
├── docker/                     ← docker-compose.yml (synced with vault)
├── docs/
│   ├── TROUBLESHOOTING.md
│   └── WORKFLOW.md
├── reports/
│   ├── samples/                ← sample report outputs
│   └── templates/              ← Jinja2/HTML templates
├── scripts/                    ← 9 scripts (canary, up, down, verify, etc.)
├── skills/                     ← symlinked from ~/.hermes/profiles/pentest/skills/
├── scope/                      ← scope templates (not per-engagement)
├── tools/                      ← tool-catalog.yaml
├── venv/                       ← Python virtualenv (was in vault)
├── index.html                  ← GitHub Pages
└── PENTEST-RUNBOOK.md
```

Rules:
- Repo = version-controlled source of truth (scripts, skills, docs, templates)
- No engagement data in repo (lives in LUKS vault)
- `engagements/zerodaybrief/` in repo deleted — merged into vault's `zerodaybrief-2026-06/`
