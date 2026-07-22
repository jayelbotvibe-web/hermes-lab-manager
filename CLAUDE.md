# CLAUDE.md — hermes-lab-manager

A **documentation and skill repo**, not runtime code. It describes the supervisor /
orchestrator-worker pattern for Hermes Agent: a `default` manager profile routes forensics
and pentest tasks to two specialized worker profiles via one-shot dispatch
(`hermes -z "<prompt>" --profile forensics|pentest`), so neither lab's tools or
conversational state pollute the other.

This is the **hub document** for the sibling repos `hermes-forensics-lab` and
`hermes-pentest-lab`. It contains no code for either.

## Commands

**None.** No build, test, or lint step; no Makefile, no package.json, no CI.

- View the site: open `index.html` in a browser (self-contained, no server needed)
- Published at `https://jayelbotvibe-web.github.io/hermes-lab-manager/`
- Install the skill: copy `skill/` to `~/.hermes/skills/devops/lab-manager/`

## Stack

No programming language. `index.html` (652 lines) is a single self-contained static page —
all CSS inline, no `<script>` tags, one external asset (`architecture.png`).
`skill/SKILL.md` is Markdown with YAML frontmatter (`name: lab-manager`, `version: 1.4.0`,
`category: devops`). `.nojekyll` means GitHub Pages serves it raw.

## Architecture

| File | Role |
|---|---|
| `README.md` (193) | Entry point — pattern, specs, both labs, vaults, dispatch |
| `index.html` (652) | Pages site, same content as README |
| `SETUP.md` (234) | 10-step bare-metal-to-dispatch replication guide |
| `PITFALLS.md` (181) | 13-entry BUG/FIX/DESIGN registry |
| `skill/SKILL.md` (150) | The actual deliverable — routing rules, dispatch verification |
| `skill/references/folder-structure.md` | Canonical layout of both labs |

## Important: this repo is deliberately sanitized

Paths use a placeholder user (`/home/user/...`) and IPs are scrubbed to `<SIFT-VM-IP>`,
`<SIFT-VM-SUBNET>`, `<VPN-provider>` — commit `1354838` was an explicit privacy scrub.

**The docs will not run as-is, and that is intentional.** The sibling repos
`hermes-lab-management-dashboard` and `hermes-desktop-widget` contain the *unsanitized*
real values. **Never copy specifics from those repos into this one.**

## Known inconsistencies — verify before repeating

- **Canary count contradicts itself.** README badge and header say 20 checks (commit
  `6345058`, "18→20"), but the README "Health Check" section and `skill/SKILL.md` still say
  **18**. Check the live canary rather than trusting either.
- **A stale link:** the README "Related Repos" table and `index.html:643` point at
  `hermes-lab-dashboard`, which was renamed to `hermes-lab-management-dashboard` on
  2026-07-09. Those URLs are dead.
- Commit `20153f5` claims to add a CI workflow, but there is **no `.github/` directory** —
  it never landed or was removed.

## Gotchas

- No env vars or services for this repo itself, but the documented workflow assumes Hermes
  v0.17.0, Docker, VMware Workstation 17.x, Ubuntu 24.04+.
- Documented pitfall worth carrying forward: **`hermes -z` has a 600s hard limit in the
  foreground**, while full engagements take 8–12 minutes. They must be background-dispatched.

## State

Complete as a docs artifact. 18 commits, all between 2026-06-27 and 2026-07-09; last was
the canary bump. Clean tree on `main`, pushed. No CI, no TODO/FIXME.

Content overlaps substantially with `hermes-lab-management-dashboard`'s
`MULTI-PROFILE-MANAGER.md` — worth deciding which is canonical if you touch either.
