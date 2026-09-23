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
scope check ──> [chapters/01-stages.md]   # stages carry class labels:
  1 subdomains   amass + sublist3r + crt.sh   [P passive]
                 gobuster dns / amass -brute  [I gated]
                 -> 01-subdomains.txt (all leads) filtered through
                    hunt/<target>/allowlist.txt -> 01-subdomains-allowlisted.txt
  2 live probe   httpx -rate-limit 50         [T target traffic]
  3 port scan    nmap top-ports (owned IPs)   [I gated — off default path;
                 masscan = exceptional, explicit permission only]
  4 dir enum     dirsearch / gobuster dir     [I gated]
  5 screenshots  gowitness v3                 [T read-only]
  6 github/pastes trufflehog + gitleaks + dorks [P — secrets stay in raw files]
  7 tech fingerprint whatweb + wappalyzer + httpx -tech-detect  [T]
     │
     └─> chapters/02-outputs.md : normalize -> diff vs prior -> new-since-last-run.txt
     └─> chapters/03-monitoring.md : schedule -> alert on diff -> repeat
```

Classes decide the gate: **P** hits third-party infra only (default-safe);
**T** sends traffic to allowlisted targets (written authorization +
allowlist-derived input + low rate); **I** is intrusive/high-volume
(explicit policy permission, owned IPs, recorded rate — never routine).

## Golden rules

1. **Allowlist-derived, deny by default.** `hunt/<target>/allowlist.txt` is
   the machine-readable scope; every target-traffic stage reads only what
   matched it. Missing/empty allowlist = nothing is in scope — discovered
   assets are leads until allowlisted + ownership-checked. Exclusions only
   subtract; they never grant.
2. **Scope gate before packets.** Read the program policy every run — and
   pin it (policy version/hash + scope snapshot into the run dir). Passive
   (P) sources are default-safe; T stages need written authorization; I
   stages need *explicit* technique permission. CDN/shared IPs never inherit
   authorization from a scoped hostname.
3. **Diff-first philosophy.** A flat asset list rots; the lead file is
   `new-since-last-run.txt` — new subdomains, new ports, new endpoints are the
   least-hunted surface. Never archive a run without diffing it.
4. **Normalize everything.** `sort -u`, lowercase domains, strip wildcards and
   schemes, one asset per line — diffs are only honest when formats are stable.
5. **Every tool writes a file.** Terminal scrollback is not an artifact. One
   output file per tool per run, in the dated directory, named for its stage.
   `raw-*` keeps tool output verbatim for forensics.
6. **Secrets stay in raw files.** trufflehog/gitleaks raw JSON contain live
   credentials — `$OUT` only, never notes/reports; leads carry location +
   type + masked prefix. Found credentials are never validated without the
   same authorization as any credential use. Purge `gh-clones/`.
7. **New assets are leads, not permission.** Four-step re-check before any
   active follow-up: ownership → allowlist → exclusions → method.
8. **Recon feeds testing.** Live hosts -> Burp scope + screenshots; tech list ->
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
mkdir -p "$OUT" "hunt/$TARGET"
# write hunt/$TARGET/allowlist.txt from the policy FIRST (deny-by-default)
for f in allowlist.txt scope-includes.txt scope-exclusions.txt scope-ip-exclusions.txt; do
  cp "hunt/$TARGET/$f" "$OUT/" 2>/dev/null || true   # snapshot scope with the run
done
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
