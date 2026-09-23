# 01-stages.md — the seven stages

Conventions used below:

```bash
TARGET=example.com                 # scoped root domain
ORG=example                        # org name for code searches
DATE=$(date +%Y%m%d)
OUT="recon/$TARGET/$DATE"          # dated run dir (see 02-outputs.md)
WL_DNS=/usr/share/seclists/Discovery/DNS/subdomains-top1million-110000.txt
WL_DIR=/usr/share/seclists/Discovery/Web-Content/raft-medium-directories.txt
mkdir -p "$OUT"
```

Every stage ends with one normalized `NN-name.txt` artifact in `$OUT` (plus
raw per-tool files for forensics). Wordlists shown are Kali SecLists paths —
adjust per machine, record the list used in `run.log`.

---

## Stage 1 — subdomain enumeration

**Purpose:** widest possible net for `*.example.com`. Passive sources first
(cert logs, search-engine indexes, amass passive), then optional brute-force
permutations. Merge everything into one deduped host list.

**Commands**

```bash
# passive: cert transparency (no packets to target)
curl -s "https://crt.sh/?q=%25.$TARGET&output=json" \
  | jq -r '.[].name_value' | tr 'A-Z' 'a-z' | sed 's/^\*\.//' \
  | sort -u > "$OUT/raw-crtsh.txt"

# passive: amass passive + sublist3r
amass enum -passive -d "$TARGET" -o "$OUT/raw-amass-passive.txt"
sublist3r -d "$TARGET" -o "$OUT/raw-sublist3r.txt"

# active (gated): DNS brute-force + amass active/alts
gobuster dns -d "$TARGET" -w "$WL_DNS" -t 50 -i -o "$OUT/raw-gobuster-dns.txt"
amass enum -active -brute -d "$TARGET" -w "$WL_DNS" -o "$OUT/raw-amass-active.txt"
```

**Normalize + merge**

```bash
cat "$OUT"/raw-crtsh.txt "$OUT"/raw-amass-passive.txt "$OUT"/raw-sublist3r.txt \
  | tr 'A-Z' 'a-z' | sed 's/^\*\.//; s/\r$//' | grep -E '\.[a-z]{2,}$' \
  | sort -u > "$OUT/01-subdomains-passive.txt"
# if active was authorized, add its hosts too:
grep -oE '[a-z0-9._-]+\.[a-z0-9.-]+' "$OUT"/raw-gobuster-dns.txt "$OUT"/raw-amass-active.txt 2>/dev/null \
  | cut -d: -f2 | tr 'A-Z' 'a-z' | sort -u > "$OUT/01b-subdomains-active.txt"
cat "$OUT"/01-subdomains-passive.txt "$OUT"/01b-subdomains-active.txt 2>/dev/null \
  | sort -u > "$OUT/01-subdomains.txt"
# drop known out-of-scope exclusions (keep a scoped copy for diffing)
if [ -s scope-exclusions.txt ]; then
  grep -vFf scope-exclusions.txt "$OUT/01-subdomains.txt" > "$OUT/01-subdomains-scoped.txt" || true
else
  cp "$OUT/01-subdomains.txt" "$OUT/01-subdomains-scoped.txt"   # no exclusions = all in scope
fi
```

**Output files:** `01-subdomains.txt` (all), `01-subdomains-scoped.txt` (in-scope),
`raw-crtsh.txt`, `raw-amass-passive.txt`, `raw-sublist3r.txt`, `raw-gobuster-dns.txt`,
`raw-amass-active.txt`.

**Failure notes**

- crt.sh returns nothing or times out -> retry, or use Cert Spotter/Censys; SAN
  lists lag weeks behind reality.
- sublist3r is abandonware-adjacent; if it crashes, its sources overlap amass's —
  proceed without it.
- amass v4 renamed flags (`enum` subcommand flags differ); check `amass enum -h`
  for `-passive`/`-active`/`-brute` on your build.
- `gobuster dns` prints `Found: host [IP]` — hence the `grep -oE` extraction above.
- `grep -vFf` treats exclusion lines as fixed strings: `api.example.com` also drops
  `test-api.example.com`, and a wildcard like `*.dev.example.com` matches nothing.
  Write exclusions as literal names, or switch to regex (`grep -vE 'dev\.example\.com$'`).
- Wildcard DNS makes every brute-forced name resolve; spot-check a random name
  (`dig +short A does-not-exist-12345.$TARGET`) and subtract the wildcard answer IPs.
  Plain `dig +short` also prints CNAME targets mid-chain — ask for `A` explicitly
  and filter to `^[0-9.]+$` when you want IPs only.

**Passive/active gating:** crt.sh, amass `-passive`, sublist3r = passive (target
sees nothing). `gobuster dns` and amass `-active -brute` query resolvers for
guessed names — low impact but technically active; run only when the policy
permits DNS brute-force, and keep the scoped/unscoped split above.

---

## Stage 2 — live probe

**Purpose:** turn the subdomain list into URLs that actually serve HTTP(S),
with status, title, tech, and IP per host. This is the working list for every
later stage.

**Commands**

```bash
httpx -l "$OUT/01-subdomains-scoped.txt" -o "$OUT/raw-httpx-urls.txt" -silent
httpx -l "$OUT/01-subdomains-scoped.txt" -sc -cl -title -tech-detect -ip -fr \
  -o "$OUT/02-live-probe.txt" -silent
```

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

**Passive/active gating:** httpx sends real HTTP requests to the target —
active. It is the cheapest active step and almost always in-scope, but it is
where "passive" ends: everything from here down touches the wire.

---

## Stage 3 — port scan

**Purpose:** find the non-HTTP surface — forgotten admin panels, dev ports,
exposed services — on owned netblocks. Two gears: masscan for breadth, nmap
for depth/service ID.

**Commands**

```bash
# resolve scoped hosts to IPs first (scans run against IPs, not URLs)
dnsx -l "$OUT/01-subdomains-scoped.txt" -a -resp-only -o "$OUT/raw-ips.txt"
# dig fallback per host: dig +short A host | grep -E '^[0-9.]+$'  (+short prints CNAMEs too)
sort -u "$OUT/raw-ips.txt" > "$OUT/03-ips.txt"

# breadth (needs root, watch the rate)
sudo masscan -iL "$OUT/03-ips.txt" -p1-65535 --rate=10000 -oL "$OUT/raw-masscan.txt"

# depth (top ports + service detection)
nmap -iL "$OUT/03-ips.txt" --top-ports 1000 -sV -oA "$OUT/raw-nmap-top1000"
# parse open ports into the canonical artifact
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
sort -u "$OUT/03-ports.txt" -o "$OUT/03-ports.txt"
```

**Output files:** `03-ips.txt`, `03-ports.txt` (`ip:port` lines), `raw-masscan.txt`,
`raw-nmap-top1000.{nmap,gnmap,xml}`.

**Failure notes**

- masscan requires root and bypasses the OS stack — a too-high `--rate` saturates
  links and looks like an attack to defenders. Start at 1k-10k, keep
  `--excludefile` for known-sensitive ranges.
- masscan output lines look like `open tcp 443 1.2.3.4 1695555555` — field order
  is `state proto port ip ts`; the awk above prints `ip:port`.
- nmap `.gnmap` is NOT line-per-port — ports are a comma-separated `port/state/proto`
  list inside the `Ports:` field of the host line. `grep '^[0-9]+/tcp'` matches
  nothing; the awk above splits `Host:`/`Ports:` fields instead.
- Hosts behind Cloudflare/Fastly/etc. answer every port open-looking or none —
  CDN ranges are usually out of scope anyway; scan owned space (ASN/netblocks),
  not CDN edges.
- nmap `-sC` (default scripts) is noisy and can trip IDS; use only with
  justification in `run.log`.
- Windows: masscan builds exist but nmap+Zenmap is the sane default.

**Passive/active gating:** this is the loudest stage in the runbook — unambiguously
active. Confirm the policy permits port scanning at all, restrict `-iL` to
in-scope owned IPs (not every IP a subdomain resolved to through a CDN), record
the rate used, and skip entirely if the program forbids it.

---

## Stage 4 — directory & file enumeration

**Purpose:** per live host, brute-force paths for admin panels, config files,
backups, old endpoints. Expensive — budget it: deep on interesting hosts,
shallow `common.txt` pass on the rest.

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

**Passive/active gating:** fully active and the most request-dense stage —
thousands of GETs per host. Requires explicit scope permission (most web-target
programs allow it; rate-limited programs may not). Never run against hosts that
failed the scoped check, and throttle on shared-infra programs.

---

## Stage 5 — screenshots

**Purpose:** eyeball every live host at once. Login portals, debug pages,
default installs, and "why does this exist" apps fall out of a 10-minute skim
that no status-code column reveals.

**Commands**

```bash
eyewitness --web -f "$OUT/02-live-hosts.txt" -d "$OUT/05-eyewitness" \
  --no-prompt --threads 8
# open the triage report
ls "$OUT/05-eyewitness/report.html"
```

Triage the report by hand; write one line per interesting host:

```bash
printf '%s\n' "https://dev.$TARGET — default Jenkins login, no auth noted" \
  >> "$OUT/05-screenshots.txt"
```

**Output files:** `05-screenshots.txt` (manual triage notes — the diffable
artifact), `05-eyewitness/` (report.html + PNGs — evidence, not diffed).

**Failure notes**

- Command name varies by distro: `eyewitness`, `EyeWitness`, or
  `python3 EyeWitness.py`. Needs geckodriver/Firefox; headless errors usually
  mean a missing driver, not dead hosts.
- Timeouts pile up on dead-ish hosts; `--timeout` (per-request) and a smaller
  input file keep runs sane. Screenshot only `02-live-hosts.txt`, not raw subs.
- Auth-gated apps screenshot as login pages — still note them; the login type
  (SSO vs local vs default-looking) is the signal.
- Alternatives if eyewitness breaks: gowitness, aquatone — same `-f url-list`
  workflow.

**Passive/active gating:** active (one GET + render per host) but read-only and
low-rate — same exposure class as the stage-2 probe you already gated. The
triage itself is offline.

---

## Stage 6 — GitHub, pastes & code leaks

**Purpose:** the org's own public footprint: hardcoded tokens in repos, internal
hostnames in gists, config dumps on pastebins. Highest lead-per-minute ratio of
any stage, and fully passive toward the target.

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
  bulk noise — sort, don't drown.
- Adjacent checks worth an hour (note results in the lead file): cloud bucket
  permutations (`$ORG-dev`, `$ORG-backup` on the usual endpoints), Google dorks
  (`site:$TARGET inurl:admin`, `ext:log|sql|conf`), Wayback for dead endpoints.

**Passive/active gating:** passive toward the target — all traffic goes to
GitHub/paste/search infrastructure. Two cautions: rate limits on the GitHub
API (use `gh`, back off on 403s), and verifying a found token against target
auth *is* active — that verification step needs the same scope check as any
credential use.

---

## Stage 7 — tech fingerprinting

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

**Passive/active gating:** `-tech-detect` rides along on the already-gated stage-2
requests. whatweb `-a 1` ≈ one request per host (same class as probing); `-a 3`
and wappalyzer's headless render are heavier — apply on the interesting-host
subset, not the whole list.
