# 01-stages.md — the seven stages

Conventions used below:

```bash
TARGET=example.com                 # scoped root domain
ORG=example                        # org name for code searches
DATE=$(date +%Y%m%d)
OUT="recon/$TARGET/$DATE"          # dated run dir (see 02-outputs.md)
export ALLOWLIST="hunt/$TARGET/allowlist.txt"     # machine-readable in-scope
                                                  # list — ONLY source T/I
                                                  # stages may draw from
export EXCLUSIONS="hunt/$TARGET/scope-exclusions.txt"
WL_DNS=/usr/share/seclists/Discovery/DNS/subdomains-top1million-110000.txt
WL_DIR=/usr/share/seclists/Discovery/Web-Content/raft-medium-directories.txt
mkdir -p "$OUT"
```

Every stage ends with one normalized `NN-name.txt` artifact in `$OUT` (plus
raw per-tool files for forensics). Wordlists shown are Kali SecLists paths —
adjust per machine, record the list used in `run.log`.

## Stage classes — three different kinds of traffic

Every stage below carries a class label. Classes, not stage numbers, decide
what permission you need:

- **P — passive third-party.** All requests go to public/third-party
  infrastructure (CT logs, search engines, GitHub, archives). The target
  sees nothing. Default-safe on any program.
- **T — target traffic.** Packets/requests reach the scoped target or its
  hosting edge. Requires written authorization AND targets drawn only from
  `allowlist.txt`. Default to low rate, read-only requests.
- **I — intrusive / high-volume.** Port scans, DNS brute-force, directory
  brute-force. Requires *explicit* policy permission (not just a scope list),
  a strict allowlist-derived target file, conservative rate, and
  shared-infrastructure exclusion. Never part of a routine run by default.

The deny-by-default rule applies inside every T/I stage: a discovered
hostname or resolved IP is a **lead** until it is confirmed on the allowlist
and owned by the program — shared/CDN IPs never inherit authorization from
a related hostname.

---

## Stage 1 — subdomain enumeration   [P passive; optional I gated]

**Purpose:** widest possible net for `*.example.com`. Passive sources first
(cert logs, search-engine indexes, amass passive), then optional brute-force
permutations. Merge everything into one deduped host list — then **filter by
allowlist membership**, not just by exclusions.

**Commands — passive [P]**

```bash
# cert transparency (no packets to target)
curl -s "https://crt.sh/?q=%25.$TARGET&output=json" \
  | jq -r '.[].name_value' | tr 'A-Z' 'a-z' | sed 's/^\*\.//' \
  | sort -u > "$OUT/raw-crtsh.txt"

# amass — VERSION MATTERS (v5 rewrote the tool):
#   v4.x (classic CLI):
amass enum -passive -d "$TARGET" -o "$OUT/raw-amass-passive.txt"
#   v5.x (engine + Asset Database): `enum` auto-starts the local engine and
#   results land in the asset DB, not a -o file (a plain `-o` can come out
#   empty). Populate, then export from the DB:
amass enum -passive -d "$TARGET"                 # engine auto-starts
amass subs -d "$TARGET" > "$OUT/raw-amass-passive.txt"   # check `amass subs -h`
#   `amass -version` tells you which family you're on; many hunters pin
#   amass v4.2.0 for the simple -o workflow — either is fine, document it.
sublist3r -d "$TARGET" -o "$OUT/raw-sublist3r.txt"   # optional: unmaintained
```

**Commands — intrusive [I] (explicit DNS-brute-force permission required)**

```bash
gobuster dns -d "$TARGET" -w "$WL_DNS" -t 50 -i -o "$OUT/raw-gobuster-dns.txt"
amass enum -active -brute -d "$TARGET" -w "$WL_DNS" -o "$OUT/raw-amass-active.txt"  # v4 flags
```

**Normalize + merge, then filter by allowlist**

```bash
cat "$OUT"/raw-crtsh.txt "$OUT"/raw-amass-passive.txt "$OUT"/raw-sublist3r.txt \
  | tr 'A-Z' 'a-z' | sed 's/^\*\.//; s/\r$//' | grep -E '\.[a-z]{2,}$' \
  | sort -u > "$OUT/01-subdomains-passive.txt"
# if intrusive stage was authorized, add its hosts too:
grep -oE '[a-z0-9._-]+\.[a-z0-9.-]+' "$OUT"/raw-gobuster-dns.txt "$OUT"/raw-amass-active.txt 2>/dev/null \
  | cut -d: -f2 | tr 'A-Z' 'a-z' | sort -u > "$OUT/01b-subdomains-active.txt"
cat "$OUT"/01-subdomains-passive.txt "$OUT"/01b-subdomains-active.txt 2>/dev/null \
  | sort -u > "$OUT/01-subdomains.txt"          # ALL discovered = leads

# ALLOWLIST membership decides what target-traffic stages may touch.
# Missing/empty allowlist => empty allowlisted list (never "everything is
# in scope"). Wildcard lines (*.example.com) match via proper wildcard
# semantics, not grep -F fixed strings:
python3 - "$OUT/01-subdomains.txt" > "$OUT/01-subdomains-allowlisted.txt" <<'PY'
import fnmatch, sys, os
try:
    pats = [l.strip().lower() for l in open(os.environ["ALLOWLIST"])
            if l.strip() and not l.startswith("#")]
except (OSError, KeyError):
    pats = []                       # no allowlist -> nothing is in scope
excl = []
ex_path = os.environ.get("EXCLUSIONS", "")
if ex_path and os.path.exists(ex_path):
    excl = [l.strip().lower() for l in open(ex_path)
            if l.strip() and not l.startswith("#")]
def hit(host, pats):
    for p in pats:
        if p.startswith("*."):
            base = p[2:]
            # *.example.com covers subdomains only; the apex needs its own
            # allowlist line — do not silently widen scope
            if host.endswith("." + base):
                return True
        elif fnmatch.fnmatch(host, p):
            return True
    return False
for line in open(sys.argv[1]):
    h = line.strip().lower()
    if h and hit(h, pats) and not hit(h, excl):
        print(h)
PY
```

Wildcard semantics written down: `*.example.com` matches
`api.example.com` and `a.b.example.com` — it does **not** cover the apex
`example.com` (the apex needs its own allowlist line; never silently widen
scope). An asset that matches no allowlist line stays in `01-subdomains.txt`
as a lead for phase-2's ownership check — it is not fed to any
target-traffic stage.

**Output files:** `01-subdomains.txt` (all leads), `01-subdomains-allowlisted.txt`
(the only list T/I stages may read), `01-subdomains-passive.txt`,
`01b-subdomains-active.txt`, `raw-*`.

**Failure notes**

- crt.sh returns nothing or times out -> retry, or use Cert Spotter/Censys; SAN
  lists lag weeks behind reality.
- sublist3r is unmaintained (last release ~2019); if it crashes, its sources
  overlap amass's — proceed without it.
- amass **v5** (2025+) is a rewrite: subcommands `engine|enum|subs|track|viz`,
  `enum` drives a local engine and stores results in the Asset Database —
  the classic `enum -o file.txt` dump does not apply (may produce an empty
  file). Export with `amass subs`/DB queries, or pin v4.2.0 for the old
  `-passive -o` flow. Check `amass -version` before assuming flags.
- `gobuster dns` prints `Found: host [IP]` — hence the `grep -oE` extraction above.
- `scope-exclusions.txt` applies on top of the allowlist through the same
  matcher — `*.dev.example.com` drops the whole subtree, `api.example.com`
  drops only that exact host. Exclusions subtract; they never widen scope.
- Wildcard DNS makes every brute-forced name resolve; spot-check a random name
  (`dig +short A does-not-exist-12345.$TARGET`) and subtract the wildcard answer IPs.
  Plain `dig +short` also prints CNAME targets mid-chain — ask for `A` explicitly
  and filter to `^[0-9.]+$` when you want IPs only.

**Gating:** crt.sh, amass passive, sublist3r = class P (target sees nothing).
`gobuster dns` and amass `-active -brute` = class I — explicit DNS-brute-force
permission, plus the allowlist filter above governs what any later stage may
touch regardless of what DNS answered.

---

## Stage 2 — live probe   [T target traffic]

**Purpose:** turn the **allowlisted** subdomain list into URLs that actually
serve HTTP(S), with status, title, tech, and IP per host. This is the working
list for every later stage.

**Commands**

```bash
# input is the ALLOWLISTED file — never 01-subdomains.txt (all leads)
httpx -l "$OUT/01-subdomains-allowlisted.txt" -o "$OUT/raw-httpx-urls.txt" -silent
httpx -l "$OUT/01-subdomains-allowlisted.txt" -sc -cl -title -tech-detect -ip -fr \
  -rate-limit 50 -o "$OUT/02-live-probe.txt" -silent
```

An empty `01-subdomains-allowlisted.txt` correctly produces an empty probe —
that is the deny-by-default outcome, not a bug.

`raw-httpx-urls.txt` is the URL list; `02-live-probe.txt` is the annotated list.
Copy the first to the canonical artifact:

```bash
cp "$OUT/raw-httpx-urls.txt" "$OUT/02-live-hosts.txt"
```

**Output files:** `02-live-hosts.txt` (one URL per line), `02-live-probe.txt`
(annotated), `raw-httpx-urls.txt`.

**Failure notes**

- `httpx` command not found or prints python-httpx usage -> the projectdiscovery
  binary may be installed as `httpx-toolkit` (Kali/Parrot naming conflict).
- Empty result on a host that answers in a browser -> WAF/CDN blocks the default
  UA or SNI; retry with `-H 'User-Agent: <browser UA>'` before declaring dead.
- `-fr` (follow redirects) catches vhosts that only respond after redirect;
  keep it, but note the final URL is what lands in the file.
- Default threads are high; add `-rate-limit 50` on programs with rate rules.

**Gating:** httpx sends real HTTP requests to the target — class T. Cheapest
target-traffic step — and it is where "passive" ends: input comes only from
`01-subdomains-allowlisted.txt`, and `-rate-limit` defaults low (50) until the
policy says otherwise.

---

## Stage 3 — port scan   [I intrusive — separate gate]

**Purpose:** find the non-HTTP surface — forgotten admin panels, dev ports,
exposed services — on **owned** netblocks. This stage is off the default run
path entirely unless the policy explicitly permits port scanning.

**Pre-flight (required):**

1. Resolve allowlisted hosts; then **ownership-check every IP** before it
   enters the scan list — an IP belonging to Cloudflare/Fastly/Akamai/
   third-party SaaS is excluded even though the hostname is in scope
   (shared infrastructure does not inherit authorization).
2. Confirm the policy permits port scanning *at all*; note the permission
   line in `run.log`.

**Commands — default (nmap, conservative)**

```bash
dnsx -l "$OUT/01-subdomains-allowlisted.txt" -a -resp-only -o "$OUT/raw-ips.txt"
# dig fallback per host: dig +short A host | grep -E '^[0-9.]+$'  (+short prints CNAMEs too)

# strip shared/CDN ranges before scanning — maintain scope-ip-exclusions.txt
# under hunt/$TARGET/ (CIDR or exact IPs of Cloudflare/Fastly/Akamai/
# managed-SaaS seen this run)
export IP_EXCLUSIONS="hunt/$TARGET/scope-ip-exclusions.txt"
python3 - "$OUT/raw-ips.txt" > "$OUT/03-ips.txt" <<'PY'
import ipaddress, sys, os
excl = []
ex_path = os.environ.get("IP_EXCLUSIONS", "")
if ex_path and os.path.exists(ex_path):
    excl = [ipaddress.ip_network(l.strip(), strict=False)
            for l in open(ex_path)
            if l.strip() and not l.startswith("#")]
for line in open(sys.argv[1]):
    ip = line.strip()
    try:
        a = ipaddress.ip_address(ip)
    except ValueError:
        continue
    if not any(a in n for n in excl):
        print(ip)
PY

nmap -iL "$OUT/03-ips.txt" --top-ports 1000 -sV -T3 --max-rate 300 \
  -oA "$OUT/raw-nmap-top1000"
```
**Commands — exceptional (masscan breadth)** — only when the policy
*explicitly* permits high-rate scanning AND the target list is owned
netblocks AND you've recorded justification + chosen rate in `run.log`:

```bash
sudo masscan -iL "$OUT/03-ips.txt" -p1-65535 --rate=1000 \
  --excludefile "hunt/$TARGET/scope-ip-exclusions.txt" -oL "$OUT/raw-masscan.txt"
```

Start at `--rate=1000`, not 10000 — masscan bypasses the OS stack and a
high rate saturates links and reads as an attack to defenders. This is the
exceptional path, never the routine one.

**Normalize**

```bash
# masscan -oL lines: "open tcp 443 1.2.3.4 <ts>" -> ip:port
grep -h 'open' "$OUT/raw-masscan.txt" 2>/dev/null | awk '{print $4":"$3}' > "$OUT/03-ports.txt"
# nmap .gnmap lines: "Host: 1.2.3.4 ()\tPorts: 22/open/tcp//ssh///, 80/open/tcp//http///, ..."
grep -h 'Ports:' "$OUT/raw-nmap-top1000.gnmap" 2>/dev/null | awk '{
  ip = $2
  plist = substr($0, index($0, "Ports:") + 6)
  n = split(plist, ent, ",")
  for (i = 1; i <= n; i++) {
    split(ent[i], f, "/")
    sub(/^[ \t]+/, "", f[1])   # entries keep a leading space after ","
    if (f[2] == "open") print ip ":" f[1]
  }
}' >> "$OUT/03-ports.txt"
touch "$OUT/03-ports.txt"      # artifact exists even when the stage is skipped
sort -u "$OUT/03-ports.txt" -o "$OUT/03-ports.txt"
```

If the stage is skipped (no port-scan permission — the common case), write a
one-line `03-ports.txt` placeholder (`# skipped: no port-scan permission`)
so the artifact contract still holds for the diff stage.

**Output files:** `03-ips.txt` (owned IPs only), `03-ports.txt` (`ip:port`),
`raw-masscan.txt` (exceptional only), `raw-nmap-top1000.{nmap,gnmap,xml}`.

**Failure notes**

- masscan requires root and bypasses the OS stack — exceptional path only;
  keep `--excludefile` pointed at `scope-ip-exclusions.txt`.
- masscan output lines look like `open tcp 443 1.2.3.4 1695555555` — field order
  is `state proto port ip ts`; the awk above prints `ip:port`.
- nmap `.gnmap` is NOT line-per-port — ports are a comma-separated `port/state/proto`
  list inside the `Ports:` field of the host line. `grep '^[0-9]+/tcp'` matches
  nothing; the awk above splits `Host:`/`Ports:` fields instead.
- Hosts behind Cloudflare/Fastly/etc. answer every port open-looking or none —
  the pre-flight IP filter is exactly what keeps CDN edges out of `-iL`.
- nmap `-sC` (default scripts) is noisy and can trip IDS; use only with
  justification in `run.log`. `-T3 --max-rate 300` is the conservative default;
  raise only if the policy tolerates it.
- Windows: masscan builds exist but nmap+Zenmap is the sane default.

**Gating:** loudest stage — class I, off the default path. Requirements stack:
written authorization + explicit port-scan permission + allowlist-derived
owned IPs only + recorded rate. Any missing item => write the `skipped`
placeholder and move on.

---

## Stage 4 — directory & file enumeration   [I intrusive — gated]

**Purpose:** per live host, brute-force paths for admin panels, config files,
backups, old endpoints. Expensive — budget it: deep on interesting hosts,
shallow `common.txt` pass on the rest. Input is `02-live-hosts.txt`, which
already descends from the allowlist — never re-derive targets here.

**Commands**

```bash
# single interesting host (dirsearch)
dirsearch -u https://dev.$TARGET -e php,html,js,txt,bak -w "$WL_DIR" -t 20 \
  -o "$OUT/raw-dirsearch-dev.$TARGET.txt" --format=plain

# or gobuster dir
gobuster dir -u https://dev.$TARGET -w "$WL_DIR" -t 20 -x php,html -k \
  -o "$OUT/raw-gobusterdir-dev.$TARGET.txt"

# shallow pass across all live hosts (small wordlist)
while read -r u; do
  n=$(echo "$u" | sed 's|https\?://||; s|[/:]|_|g')
  dirsearch -u "$u" -w /usr/share/seclists/Discovery/Web-Content/common.txt \
    -t 15 --format=plain -o "$OUT/raw-dirsearch-$n.txt"
done < "$OUT/02-live-hosts.txt"

# canonical artifact: status-bearing hits only, one per line
grep -hE '^\[?[0-9]{3}' "$OUT"/raw-dirsearch-*.txt 2>/dev/null \
  | sort -u > "$OUT/04-dirs.txt"
```

**Output files:** `04-dirs.txt`, `raw-dirsearch-<host>.txt`, `raw-gobusterdir-<host>.txt`.

**Failure notes**

- Wildcard/soft-404 sites return 200 for everything -> both tools support
  auto-calibration (dirsearch `--exclude-status` / gobuster `-b` blacklist codes);
  verify a nonsense path by hand first.
- 429s or escalating response times = rate limiting; drop `-t`, add delays, or
  stop — hammering login-adjacent paths can trip lockout controls.
- Runtime is per-host x wordlist size; a full `raft-medium` against 200 hosts is
  days. Prioritize: new hosts, dev/staging names, API-looking hosts.
- dirsearch report flag changed across versions: `-o FILE --format=plain` on
  current releases; older builds used `--plain-text-report=FILE`.
- `-k` on gobuster ignores TLS errors — needed for dev/staging certs, but note
  that you used it.

**Gating:** class I — most request-dense stage, thousands of GETs per host.
Requires explicit scope permission (most web-target programs allow it;
rate-limited programs may not). Targets come only from `02-live-hosts.txt`;
throttle `-t` and add delays on shared-infra or rate-sensitive programs.

---

## Stage 5 — screenshots   [T target traffic — read-only]

**Purpose:** eyeball every live host at once. Login portals, debug pages,
default installs, and "why does this exist" apps fall out of a 10-minute skim
that no status-code column reveals.

**Commands — gowitness v3 (maintained)**

```bash
gowitness scan file -f "$OUT/02-live-hosts.txt" \
  --write-db --screenshot-path "$OUT/05-gowitness"
# triage in the report viewer (gowitness report server), or browse PNGs
ls "$OUT/05-gowitness"
```

Triage by hand; write one line per interesting host:

```bash
printf '%s\n' "https://dev.$TARGET — default Jenkins login, no auth noted" \
  >> "$OUT/05-screenshots.txt"
```

**Output files:** `05-screenshots.txt` (manual triage notes — the diffable
artifact), `05-gowitness/` (screenshots + sqlite DB — evidence, not diffed).

**Failure notes**

- gowitness **v3** rewrote the CLI: `scan file -f LIST --write-db
  --screenshot-path DIR`. The v2 `gowitness file -f` / `single URL` forms are
  gone — check `gowitness --help` on your build. `report server` serves the
  DB; without `--write-db` only PNGs land on disk.
- Needs a Chrome/Chromium it can drive (headless CDP) — missing browser is
  the usual failure, not dead hosts.
- Timeouts pile up on dead-ish hosts; a smaller input file keeps runs sane.
  Screenshot only `02-live-hosts.txt`, not raw subs.
- Auth-gated apps screenshot as login pages — still note them; the login type
  (SSO vs local vs default-looking) is the signal.
- eyewitness/aquatone still work if you have them but are less maintained;
  the `-f url-list` workflow is equivalent.

**Gating:** class T (one GET + render per host), read-only, low rate — same
exposure as the stage-2 probe it reads from. The triage itself is offline.

---

## Stage 6 — GitHub, pastes & code leaks   [P passive — secrets handling]

**Purpose:** the org's own public footprint: hardcoded tokens in repos, internal
hostnames in gists, config dumps on pastebins. Highest lead-per-minute ratio of
any stage, and fully passive toward the target.

**Secret-handling rules (non-negotiable):**

- Raw scanner output (`raw-trufflehog.json`, `raw-gitleaks-*.json`, cloned
  repos) **contains live secrets**. Keep it inside `$OUT` only — never paste
  into `notes.md`, evidence bundles, or reports. Reports cite
  file-path + key-type + a masked prefix (`AKIA…AB12` style), never the value.
- `06-code-leads.txt` carries locations only (repo, file, secret type) — no
  secret material.
- A found credential is never validated against target auth without the
  same explicit authorization as any credential use (see the orchestrator's
  scope contract). Enumeration ≠ permission to log in.
- `gh-clones/` is a cache holding leaked secrets in plain text — delete it
  after extraction (`rm -rf "$OUT/gh-clones"`) once leads are recorded.

**Commands**

```bash
# enumerate org repos, clone for local searching
gh repo list "$ORG" --limit 500 --json nameWithOwner --jq '.[].nameWithOwner' \
  > "$OUT/raw-gh-repos.txt"
mkdir -p "$OUT/gh-clones" && cd "$OUT/gh-clones"
while read -r r; do gh repo clone "$r" -- --depth 50; done < "../raw-gh-repos.txt"
cd - >/dev/null

# secret scanning (finds committed keys/tokens/config)
trufflehog github --org="$ORG" --results=verified --json > "$OUT/raw-trufflehog.json"
for d in "$OUT"/gh-clones/*/; do
  gitleaks detect --source "$d" --report-path "$OUT/raw-gitleaks-$(basename "$d").json"
done

# manual dorks (browser): run these, save hits by hand
#   org:example "internal.example.com"
#   org:example filename:.env OR filename:config OR filename:docker-compose.yml
#   org:example "password" extension:yml
#   site:pastebin.com "$TARGET"      site:ideone.com|site:codepad.org "$TARGET"
```

`raw-trufflehog.json` / `raw-gitleaks-*.json` are JSONL/JSON; reduce to a
diffable lead list:

```bash
jq -r '.SourceMetadata.Data.Github.repository // .SourceMetadata.Data.Git.repository // empty' \
  "$OUT"/raw-trufflehog.json 2>/dev/null | sort -u > "$OUT/06-code-leads.txt"
jq -r '.[].File' "$OUT"/raw-gitleaks-*.json 2>/dev/null | sort -u >> "$OUT/06-code-leads.txt"
sort -u "$OUT/06-code-leads.txt" -o "$OUT/06-code-leads.txt"
# append manual dork hits as "DORK <query> -> <where>" lines
```

**Output files:** `06-code-leads.txt`, `raw-gh-repos.txt`, `raw-trufflehog.json`,
`raw-gitleaks-*.json`, `gh-clones/` (cache — delete after extraction if space
matters).

**Failure notes**

- trufflehog v3 syntax is `trufflehog git <url>` / `trufflehog github --org=X`;
  the v2 `--regex --entropy=True URL` form is gone. There is no `-o` — `--json`
  prints one JSONL result per line on stdout, so redirect with `>`. Repo names
  live at `.SourceMetadata.Data.Github.repository` (`.Data.Git.repository` for
  `trufflehog git` output) — not `.SourceMetadata.Github.Repository`.
- gitleaks `--report-path` writes a top-level JSON *array* — parse with `.[].File`,
  not `.File`.
- gitleaks needs the git history (`--depth` still sees recent commits; drop the
  flag for full history on high-value repos).
- GitHub code search needs auth — `gh auth login` or an API token; unauthenticated
  web search misses `filename:`/`extension:` qualifiers.
- Verified findings from trufflehog are the priority queue; unverified hits are
  bulk noise — sort, don't drown. Verified ≠ permission to use the credential.
- Adjacent checks worth an hour (note results in the lead file): cloud bucket
  permutations (`$ORG-dev`, `$ORG-backup` on the usual endpoints), Google dorks
  (`site:$TARGET inurl:admin`, `ext:log|sql|conf`), Wayback for dead endpoints.

**Gating:** class P toward the target — all traffic goes to GitHub/paste/
search infrastructure. Two cautions: rate limits on the GitHub API (use `gh`,
back off on 403s), and the credential-validation rule above.

---

## Stage 7 — tech fingerprinting   [T target traffic — low rate]

**Purpose:** stack per live host — server, framework, CMS, JS libs. Output feeds
the vuln-class checklist: WordPress -> wpscan, GraphQL -> introspection,
Java + specific versions -> CVE search.

**Commands**

```bash
# httpx already collected it in stage 2 — split it out for the artifact
# (bracket groups also hold [200] status codes and [1.2.3.4] IPs — filter them)
grep -oE '\[[a-zA-Z0-9._ /-]+\]' "$OUT/02-live-probe.txt" | tr -d '[]' \
  | grep -vE '^[0-9]{3}$|^[0-9]{1,3}(\.[0-9]{1,3}){3}$' \
  | sort -u > "$OUT/raw-tech-tags.txt"

# whatweb: aggressive level 3 over the live list
whatweb -i "$OUT/02-live-hosts.txt" -a 3 --log-brief="$OUT/raw-whatweb.txt"

# wappalyzer per interesting host (npm CLI; also a browser ext for manual checks)
npx wappalyzer https://dev.$TARGET > "$OUT/raw-wappalyzer-dev.json"

# canonical artifact: "host : tech1, tech2" lines for diffing
paste -d'|' "$OUT/02-live-hosts.txt" <(whatweb -i "$OUT/02-live-hosts.txt" -a 1 \
  --log-brief=/dev/stdout 2>/dev/null | sed 's/.*http/ http/') 2>/dev/null \
  | sed 's/|\s*/ : /' | sort -u > "$OUT/07-tech.txt"
```

(If the `paste` pipeline feels fragile, the honest fallback: `raw-whatweb.txt`
plus `raw-tech-tags.txt` are both diffable — copy whichever is cleaner to
`07-tech.txt`.)

**Output files:** `07-tech.txt`, `raw-tech-tags.txt`, `raw-whatweb.txt`,
`raw-wappalyzer-<host>.json`.

**Failure notes**

- `httpx -tech-detect` already covers most fingerprinting — stage 7 exists to
  deepen and corroborate it, not to duplicate effort.
- whatweb `-a 3` sends extra requests (aggressive); `-a 1` is passive-ish
  (one request). Use `-a 3` on interesting hosts only.
- The wappalyzer npm CLI drives a headless browser — first run downloads one;
  CI/cron boxes need the browser deps installed or it hangs silently. The
  browser extension on a manual walkthrough is the zero-setup fallback.
- CDNs front real stacks — `Akamai`/`cloudflare` tags mask origin tech; corroborate
  with error pages, cookie names, and `Server` headers in `02-live-probe.txt`.

**Gating:** class T. `-tech-detect` rides along on the already-gated stage-2
requests. whatweb `-a 1` ≈ one request per host (same class as probing); `-a 3`
and wappalyzer's headless render are heavier — apply on the interesting-host
subset, not the whole list.

---

## Edge scenarios — decide before they happen

- **No allowlist file / empty allowlist** → `01-subdomains-allowlisted.txt`
  comes out empty; every T/I stage has zero targets and the run is a passive
  collection only. That is the intended deny-by-default outcome — fix the
  scope files, never "work around" the empty list.
- **Apex vs wildcard** → `*.example.com` does not cover `example.com`; both
  need their own allowlist lines. Check `scope.md`'s written interpretation.
- **Wildcard DNS** → stage-1 note covers detection; subtract wildcard answer
  IPs before trusting any resolution.
- **CDN/shared-IP resolution** → hostname in scope, IP belongs to Cloudflare
  → IP goes to `scope-ip-exclusions.txt`; stage 3 never touches it. Scan
  owned space only.
- **429s / rising latency anywhere** → rate limiting or WAF response: drop
  threads, add delay, or stop the stage. Pushing through is a program-violation
  pattern, not persistence.
- **Secret found in stage 6** → record location + type + masked prefix in
  leads; do not validate it; purge `gh-clones/`. The credential rule in the
  orchestrator contract governs what happens next.
- **Tool version mismatch** → amass v5 vs v4 and gowitness v3 vs v2 syntax
  both differ per install; `-version`/`--help` first, record the version in
  `run.log`.
