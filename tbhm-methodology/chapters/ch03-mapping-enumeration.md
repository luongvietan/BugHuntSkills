# Ch3: Mapping & Enumeration

> **Volume note:** smart directory brute-force and spidering are volumetric — policy rate limits and technique bans apply; consolidate into few, well-chosen lists. Platform-ID and status-code escalation reasoning is durable.

Source: `03_Mapping` (+ the `v4/` wordlist payload). Mapping converts the discovered host list into a content map + technology profile — what exists, what it runs, and what it's already known to be vulnerable to.

## Mapping toolbox

| Task | Tools / resources |
|---|---|
| Content discovery | "Smart" directory brute-forcing: **RAFT lists** (in SecLists), **SVN Digger** and **Git Digger** wordlists (exposed VCS dirs) |
| Platform identification | Wappalyzer & BuiltWith (browser extensions), **retire.js** (CLI/Burp — flags JS libs with known CVEs), then check CVEs for the fingerprinted versions |
| CMS-specific enumeration | WPScan, CMSmap |
| Large-scope metadata | Bugcrowd "Maps" project (~250+ programs: crawl + DNS info + brute-force + bounty metadata → feeds Intrigue) |
| OSINT framework | **Intrigue** (`intrigueio/intrigue-core`): DNS subdomain brute-force, web spider, Nmap scan, pluggable |
| Wordlists | The repo ships `v4/all2.txt` — a ~1.3M-line merged content-discovery wordlist; pair with stack-matched lists (Assetnote/SecLists) |

## Directory brute-force workflow — status codes are a map

After the brute-force, don't discard non-200s. Codes indicating denial/auth (`401`, `403`) mark directories worth *recursing into* — misconfigured access control often protects the parent but not children:

```
GET acme.com/                  → 200
GET acme.com/backlog/          → 404   (dead)
GET acme.com/controlpanel/     → 401   (interesting — brute-force INSIDE it)
GET acme.com/controlpanel/<wordlist>  → hunt for the page they forgot to protect
```

Same trick applies to 403 "forbidden" roots and to vhost-specific paths: the auth check may exist at the front door but not on every route behind it.

## OSINT vuln history — stand on prior disclosures

Find previously reported/existing problems and let them guide you:

- Historical XSS mirrors/databases of the era: Xssed, Reddit r/xss, Punkspider, xss.cx, xssposed, Twitter search
- Issues may already be reported/fixed — but the **flaw area and injection type tell you where to hunt next**: the dev's filter style repeats, adjacent params usually share the same bug, and patched spots invite bypass attempts.

## Crawl-driven lead mining

Maps-project style: crawl target → JSON → grep for lead classes:

```bash
cat target_crawl.json | grep redirect
# → /redirect/?url=...  (open redirect / SSRF candidates)
# grep for: redirect, url, file, path, token, key, debug, api, upload
```

Every param name the crawl surfaces is a tactical-fuzzing hypothesis for ch05–ch07.
