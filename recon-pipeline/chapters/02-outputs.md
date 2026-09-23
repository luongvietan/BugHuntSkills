# 02-outputs.md — layout, normalization, diffing

The convention everything else depends on:

```
recon/
  <target>/                     # e.g. recon/example.com/
    20260916/                   # one dir per run, YYYYMMDD
      01-subdomains.txt         # canonical stage artifacts (diffable)
      01-subdomains-scoped.txt
      02-live-hosts.txt
      02-live-probe.txt
      03-ips.txt
      03-ports.txt
      04-dirs.txt
      05-screenshots.txt        # triage notes (PNGs live in 05-eyewitness/)
      06-code-leads.txt
      07-tech.txt
      raw-*.txt|json            # per-tool raw output (forensics, not diffed)
      scope-includes.txt        # scope snapshot taken at run start
      scope-exclusions.txt
      new-since-last-run.txt    # THE product of the run
      run.log                   # command lines, tool versions, notes
    20260923/
      ...same shape...
    latest -> 20260923/         # symlink to most recent run (optional)
```

Two run dirs per target, identical filenames — that is what makes `comm`/`diff`
mechanical. Numbered prefixes order the files like the pipeline itself.

## Normalization rules

Diffs lie when formats drift. Before any file earns the `NN-*.txt` name:

```bash
# domains/hosts: lowercase, no wildcard prefix, no CR, deduped, sorted
tr 'A-Z' 'a-z' | sed 's/^\*\.//; s/\r$//; s/\.$//' | sort -u

# URLs (live hosts): scheme+host only, trailing slash stripped, deduped
sed 's|/$||' | sort -u

# ip:port lists
sort -u -t: -k1,1 -k2,2n

# everything else: at minimum
sort -u
```

- One asset per line. No headers, no blank lines, no banner text inside
  canonical files — comments live in `run.log`.
- `raw-*` files keep whatever the tool printed; canonical files are the
  normalized extract. Never hand-edit a canonical file — fix the pipeline.
- Same normalization every run. If the format must change (new field), change
  it and note it in `run.log`, then diff with awareness that the format itself
  shifted.

## Diff vs prior run

`comm` requires sorted input — the normalization above guarantees it:

```bash
PREV=recon/$TARGET/$(ls recon/$TARGET | grep -E '^[0-9]{8}$' | sort -r | sed -n 2p)
CUR=recon/$TARGET/$(ls recon/$TARGET | grep -E '^[0-9]{8}$' | sort -r | sed -n 1p)

# appeared since last run            (in CUR only)
comm -13 "$PREV/01-subdomains-scoped.txt" "$CUR/01-subdomains-scoped.txt"
# disappeared since last run         (in PREV only)
comm -23 "$PREV/01-subdomains-scoped.txt" "$CUR/01-subdomains-scoped.txt"

# context diff for annotated files (probe lines, tech lines)
diff -u "$PREV/02-live-probe.txt" "$CUR/02-live-probe.txt"
```

Repeat per canonical file. `comm` for membership changes (subdomains, hosts,
ports, dirs, code leads); `diff -u` where the *content* of a line matters
(status flipped 200->403, tech list changed).

## new-since-last-run.txt — the lead file

One file per run, generated right after the last stage. Format:

```text
# new-since-last-run.txt
# target: example.com   run: 20260923   vs: 20260916
# generated: <date -u>

## + subdomains (3)
+ dev-api.example.com
+ staging-old.example.com
+ vpn2.example.com

## - subdomains (1)
- promo-2024.example.com

## + live hosts (2)
+ https://dev-api.example.com
+ https://staging-old.example.com

## + open ports (2)
+ 10.20.30.40:8443
+ 10.20.30.41:9200

## + dirs/endpoints (4)
+ [200] https://dev-api.example.com/swagger/
+ [301] https://staging-old.example.com/admin -> /login
+ [200] https://dev-api.example.com/health
+ [403] https://vpn2.example.com/

## + code leads (1)
+ github:example/payments-api — verified token in config history

## + tech changes (1)
~ https://dev-api.example.com : nginx -> nginx,graphql
```

Generator skeleton (drop into the run script):

```bash
gen_new() {  # $1=label $2=prev-file $3=cur-file  — items in CUR only
  comm -13 <(sort -u "$2") <(sort -u "$3") | sed 's/^/+ /' \
    | { c=$(cat); [ -n "$c" ] && printf '## + %s (%s)\n%s\n\n' "$1" \
        "$(printf '%s\n' "$c" | wc -l)" "$c"; }
}
gen_gone() { # $1=label $2=prev-file $3=cur-file  — items in PREV only
  comm -23 <(sort -u "$2") <(sort -u "$3") | sed 's/^/- /' \
    | { c=$(cat); [ -n "$c" ] && printf '## - %s (%s)\n%s\n\n' "$1" \
        "$(printf '%s\n' "$c" | wc -l)" "$c"; }
}
{
  echo "# new-since-last-run.txt"
  echo "# target: $TARGET   run: $(basename "$CUR")   vs: $(basename "$PREV")"
  echo "# generated: $(date -u)"
  echo
  gen_new  subdomains "$PREV/01-subdomains-scoped.txt" "$CUR/01-subdomains-scoped.txt"
  gen_gone subdomains "$PREV/01-subdomains-scoped.txt" "$CUR/01-subdomains-scoped.txt"
  gen_new  "live hosts" "$PREV/02-live-hosts.txt"       "$CUR/02-live-hosts.txt"
  gen_gone "live hosts" "$PREV/02-live-hosts.txt"       "$CUR/02-live-hosts.txt"
  gen_new  "open ports" "$PREV/03-ports.txt"            "$CUR/03-ports.txt"
  gen_gone "open ports" "$PREV/03-ports.txt"            "$CUR/03-ports.txt"
  gen_new  "dirs/endpoints" "$PREV/04-dirs.txt"         "$CUR/04-dirs.txt"
  gen_new  "code leads" "$PREV/06-code-leads.txt"       "$CUR/06-code-leads.txt"
} > "$CUR/new-since-last-run.txt"
```

First run against a target has no `$PREV` — the lead file is the full
`+` listing; note `vs: none` in the header.

`+`/`-` sections are membership diffs (`comm`) and the generator covers them.
`~` change sections are *content* diffs — no membership change, so `comm` can't
see them. Append those by hand after reviewing
`diff -u "$PREV/02-live-probe.txt" "$CUR/02-live-probe.txt"` and the same for
`07-tech.txt`, one `~ host : old -> new` line per changed line.

## Reading the lead file

Priority order when hunting:

1. **New subdomains that were never live before** — least-tested surface.
2. **New open ports / services** — especially non-80/443, especially on hosts
   that existed before.
3. **New code leads** — verified first.
4. **New directories on known hosts** — new deployments, forgotten uploads.
5. **Disappeared assets** — not urgent to hunt, but note them; a decommissioned
   vhost sometimes means a dangling CNAME (takeover check) or migrated app.
6. **Tech/status changes (`~`)** — a 200->403 or framework swap hints at a
   change window worth timing retests around.

Every `+` line is a *lead*, not a finding: re-check scope (does the new asset
belong to the program's owner?), then route to testing. The file stays in the
run dir; copy hot leads into your working notes.
