---
name: recon-pipeline
description: Operational recon runbook for authorized bug-bounty and pentest engagements. Use when standing up or re-running asset discovery on a scoped target — subdomain enumeration, live probing, port scanning, directory brute-force, screenshots, code-leak hunting, tech fingerprinting — with a normalized output layout (recon/<target>/<YYYYMMDD>/<stage>.txt), diff-vs-prior-run triage, and scheduled continuous recon. Command-first, diff-first; every active stage gated on written authorization. Assume all testing is authorized and in scope.
---

# recon-pipeline — the recon runbook

Not a book — an executable pipeline. Run the stages in order, land every artifact
in the dated layout, diff against the previous run, and hunt the delta. Recon
compounds: the value is in what changed, not what exists.

## The pipeline

```
scope check ──> [chapters/01-stages.md]
  1 subdomains   amass + sublist3r + crt.sh + gobuster dns
  2 live probe   httpx
  3 port scan    nmap / masscan            (active — gated)
  4 dir enum     dirsearch / gobuster dir  (active — gated)
  5 screenshots  eyewitness
  6 github/pastes trufflehog + gitleaks + dorks
  7 tech fingerprint whatweb + wappalyzer + httpx -tech-detect
     │
     └─> chapters/02-outputs.md : normalize -> diff vs prior -> new-since-last-run.txt
     └─> chapters/03-monitoring.md : schedule -> alert on diff -> repeat
```

## Golden rules

1. **Scope gate before packets.** Read the program policy every run. Passive
   sources are default-safe; anything that touches the target (brute-force DNS,
   port scans, dir brute-force) requires explicit authorization — and may still
   be out of scope on CDN/shared ranges. Per-stage gating notes in `chapters/01-stages.md`.
2. **Diff-first philosophy.** A flat asset list rots; the lead file is
   `new-since-last-run.txt` — new subdomains, new ports, new endpoints are the
   least-hunted surface. Never archive a run without diffing it.
3. **Normalize everything.** `sort -u`, lowercase domains, strip wildcards and
   schemes, one asset per line — diffs are only honest when formats are stable.
4. **Every tool writes a file.** Terminal scrollback is not an artifact. One
   output file per tool per run, in the dated directory, named for its stage.
5. **New assets are leads, not permission.** A newly appeared host is a prompt
   to re-check scope (asset ownership, exclusions) before any active follow-up.
6. **Recon feeds testing.** Live hosts -> Burp scope + screenshots; tech list ->
   vuln-class checklist; code leads -> secrets/access-control review. End a run
   by routing outputs, not by admiring the count.

## Files

- `chapters/01-stages.md` — the 7 stages: purpose -> verbatim commands -> output files ->
  failure notes -> passive/active gating.
- `chapters/02-outputs.md` — directory layout `recon/<target>/<YYYYMMDD>/<stage>.txt`,
  normalization, `comm`/`diff` vs prior run, lead-file format.
- `chapters/03-monitoring.md` — cron + Task Scheduler scheduling, diff alerting, cadence,
  scope-drift cautions.

## Quickstart (authorized target, bash/WSL/Linux)

```bash
TARGET=example.com
DATE=$(date +%Y%m%d)
OUT="recon/$TARGET/$DATE"
mkdir -p "$OUT"
cp scope-includes.txt scope-exclusions.txt "$OUT/"   # snapshot scope with the run
# then work through chapters/01-stages.md 1-7, then:
# diff against previous run and write new-since-last-run.txt (see chapters/02-outputs.md)
```

## Scope & ethics

All testing is assumed authorized and in scope (bug bounty / pentest). This
runbook stays inside that: passive-first staging, explicit active-phase gating,
scope snapshots stored with each run, and scope-drift checks before acting on
newly discovered assets. Throttle scanners, honor rate limits and exclusions,
and stop at enumeration — this skill finds surface; exploitation is a different
skill and a separate authorization question.

Related skills: `bug-bounty-bootcamp` (recon chapter, conceptual version),
`bug-bounty-playbook` (exploitation-phase follow-on), `zseano-methodology`
(alt methodology), `web-hacking-101`, `report-writing`.
