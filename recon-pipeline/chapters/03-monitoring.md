# 03-monitoring.md — continuous recon

One-shot recon rots. The point of the dated layout + diff convention is a loop:
run on a schedule, alert only on delta, hunt the leads, repeat.

## The run script

Wrap stages 1-2 (+gated 3-7) so the schedule has one entry point:

```bash
#!/usr/bin/env bash
# run-recon.sh <target> [--active]   — passive stages always; active behind flag
set -euo pipefail
TARGET="$1"; ACTIVE="${2:-}"
BASE="$(cd "$(dirname "$0")" && pwd)"; cd "$BASE"
DATE=$(date +%Y%m%d); OUT="recon/$TARGET/$DATE"; mkdir -p "$OUT"
cp scope-includes.txt scope-exclusions.txt "$OUT/" 2>/dev/null || true
exec > >(tee -a "$OUT/run.log") 2>&1
echo "[run $DATE target=$TARGET active=${ACTIVE:-no}]"

# stage 1 (passive half always; brute-force only when --active)
curl -s "https://crt.sh/?q=%25.$TARGET&output=json" \
  | jq -r '.[].name_value' | tr 'A-Z' 'a-z' | sed 's/^\*\.//' | sort -u > "$OUT/raw-crtsh.txt"
amass enum -passive -d "$TARGET" -o "$OUT/raw-amass-passive.txt" || true
cat "$OUT/raw-crtsh.txt" "$OUT/raw-amass-passive.txt" | sort -u > "$OUT/01-subdomains.txt"
if [ -s scope-exclusions.txt ]; then
  grep -vFf scope-exclusions.txt "$OUT/01-subdomains.txt" > "$OUT/01-subdomains-scoped.txt" || true
else
  cp "$OUT/01-subdomains.txt" "$OUT/01-subdomains-scoped.txt"
fi

# stage 2 (gated: only with --active)
if [ "$ACTIVE" = "--active" ]; then
  httpx -l "$OUT/01-subdomains-scoped.txt" -silent -o "$OUT/02-live-hosts.txt"
  # stages 3-7 here, behind the same flag
fi

# diff + lead file (see 02-outputs.md gen_new/gen_gone)
ln -sfn "$OUT" "recon/$TARGET/latest"
echo "[done]"
```

`chmod 700 run-recon.sh` (private — it encodes your targets).

## Scheduling

**cron (Linux/WSL cron):**

```cron
# passive sweep weekly, Mondays 03:00 — always safe to schedule
0 3 * * 1  /home/you/recon/run-recon.sh example.com >> /home/you/recon/cron.log 2>&1
# active run monthly, 1st at 04:00 — ONLY if the policy allows scheduled scanning
0 4 1 * *  /home/you/recon/run-recon.sh example.com --active >> /home/you/recon/cron.log 2>&1
```

Notes: cron has no missed-job catch-up — a laptop asleep at 03:00 skips the run
(use `anacron` or a systemd timer if that matters). WSL cron needs the service
running; on Windows hosts prefer Task Scheduler:

**Task Scheduler (Windows, driving WSL):**

```bat
schtasks /create /tn "recon-example-passive" /sc weekly /d MON /st 03:00 ^
  /tr "wsl -e bash -lc '/home/you/recon/run-recon.sh example.com'"
schtasks /create /tn "recon-example-active" /sc monthly /d 1 /st 04:00 ^
  /tr "wsl -e bash -lc '/home/you/recon/run-recon.sh example.com --active'"
```

GUI equivalent: Task Scheduler -> Create Task -> Triggers (weekly/monthly) ->
Action `wsl.exe` with args `-e bash -lc '/home/you/recon/run-recon.sh example.com'`,
plus "Run whether user is logged on or not" and "Wake the computer to run" off
for a laptop.

## Diff alerting

Alert on the lead file's existence-with-content — zero noise when nothing
changed. Tail of `run-recon.sh`:

```bash
LEAD="$OUT/new-since-last-run.txt"
# ... generate it (02-outputs.md) ...
if grep -q '^+ ' "$LEAD"; then
  COUNT=$(grep -c '^+ ' "$LEAD")
  MSG="recon $TARGET: $COUNT new assets — $LEAD"
  # pick one channel:
  mail -s "$MSG" you@example.com < "$LEAD"                          # local mail
  notify-send "recon" "$MSG"                                       # desktop
  curl -s -X POST -H 'Content-type: application/json' \
    --data "{\"text\":\"$MSG\"}" "$SLACK_HOOK" >/dev/null          # slack hook
fi
```

Keep the hook URL in an env var sourced from a private file, not in the script —
it grants a channel into your chat, treat it like a credential.

Alert-fatigue guard: alert on `+` lines only, never on `-` or `~` alone; if a
run produced zero leads, the artifact in the dated dir is the record — silence
is a valid result.

## Cadence

| Data | Suggested cadence | Why |
|---|---|---|
| crt.sh / passive subs | weekly | cert logs lag days; weekly catches them cheaply |
| live probe + screenshots | weekly (with passive run if allowed) or monthly | cheap; host churn is the lead engine |
| port scan | monthly, or on new-IP delta only | loud; diff-triggered beats calendar-triggered |
| dir enum | on new-host delta, or monthly on stable hosts | expensive; prioritize delta |
| GitHub/code | continuous (watch repo events) or weekly | pushes land anytime; first-to-scan wins |
| tech fingerprint | with every probe run | nearly free, high signal on `~` lines |

Rule of thumb: passive on a clock, active on a delta. A monthly active run that
always finds nothing is a signal to loosen it, not tighten it.

## Scope-drift cautions

Continuous recon discovers assets faster than programs update policies. Before
acting on any `+` line:

- **Re-read the policy.** Scope, exclusions, and allowed-techniques change;
  the snapshot in the run dir tells you what scope was *at run time*, not now.
- **New subdomain != in scope.** Verify ownership (whois/ASN, cert CN, corp
  infra vs third-party SaaS). `*.example.com` in-scope does not make
  `example.zendesk.com` or `example.statuspage.io` yours to test.
- **Wildcard != permission.** `*.example.com` scope still respects the exclusion
  list — keep `scope-exclusions.txt` current and re-apply it before every
  active stage.
- **Acquisitions drift.** Programs add/remove brands; a target set that was
  correct in June can include a divested domain by September. Diff scope
  snapshots too: `diff recon/$TARGET/<old>/scope-includes.txt recon/$TARGET/<new>/scope-includes.txt`.
- **Rate and technique limits.** Some programs cap req/sec or ban scanners
  outright on certain assets — re-check before scheduling `--active`.
- **Data hygiene.** Runs accumulate screenshots and code-lead JSON for real
  targets: keep `recon/` private (`chmod -R 700`), encrypt at rest on shared
  machines, and purge `gh-clones/` after extraction.
- **Stop conditions.** If a scheduled run ever produces traffic outside scope
  (DNS shows the domain moved to a third party, policy pulled the wildcard),
  disable the job and annotate `run.log` — an automated task that outlives its
  authorization is an incident, not a bug.
