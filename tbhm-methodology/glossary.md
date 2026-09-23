# Glossary — The Bug Hunter's Methodology (TBHM)

Terms as Haddix uses them across the TBHM docs and talks. Dated-tool entries are kept for fidelity; see `recon-pipeline` for the modern stack.

## Program & process

- **Single-sourced testing** — traditional one-tester engagement: common vulns, guaranteed pay, no competition.
- **Crowdsourced testing** — bug bounty model: raced against other hunters, paid on impact/uniqueness, duplicates worth $0.
- **1st-party program** — run by the vendor itself (Google, PayPal). **2nd-party** — platform-mediated (Bugcrowd, HackerOne, Synack).
- **Wildcard scope** — `^.acme.com`-style policy covering all subdomains; the discovery stage's best friend.
- **Acquisition rules** — program policy on whether/when acquired companies are in scope (e.g., grace periods post-acquisition).
- **Report template** — pre-built writeup skeleton for recurring vuln classes; the speed edge in raced bounties.
- **Attack scenario** — the narrative of what an attacker gains; the lever that moves severity, not the raw mechanism.

## Discovery & mapping

- **Road less traveled** — the core TBHM discovery principle: hunt assets/features others skip (acquisitions, odd ports, mobile sites, redesigns).
- **Recon-ng** — Haddix-era OSINT recon framework; `enumall` was his domain-enumeration automation script.
- **Intrigue** — OSINT framework (intrigueio) wrapping DNS subdomain brute-force, spidering, Nmap, pluggable tasks.
- **Maps project** — Bugcrowd Labs' crawl+DNS+brute-force+bounty-metadata dataset (~250+ programs) feeding Intrigue.
- **RAFT lists** — classic large directory/file wordlists (bundled in SecLists).
- **SVN Digger / Git Digger** — wordlists for exposed `.svn`/`.git` repository files.
- **all2.txt (`v4/`)** — TBHM's ~1.3M-line merged content-discovery wordlist; pair with stack-matched lists.
- **retire.js** — scanner for JS libraries with known vulnerabilities (CLI or Burp plugin).
- **WPScan / CMSmap** — CMS-specific enumerators (WordPress / multi-CMS).
- **OSINT vuln mirrors** — historical disclosure archives (Xssed, Punkspider, xssposed, r/xss, Twitter search) used to predict flaw areas; mostly defunct, modern equivalent = HackerOne hacktivity + disclosed reports.
- **Status-code escalation** — recursing brute-force into `401`/`403` directories to find routes behind misconfigured access control.

## Testing concepts

- **Tactical fuzzing** — TBHM's 80/20 approach: polyglot first, targeted payloads only where it lands; driven by per-feature core questions.
- **Polyglot payload** — single string valid in multiple contexts (HTML/JS/SQL quote contexts), used as a smoke test.
- **SWF parameter XSS** — Flash-era injection via `callback`/`xmlPath`-style params; modern analog = JS-component params and postMessage handlers.
- **Fake param injection** — appending nonexistent parameters carrying payloads; many apps reflect unknown inputs.
- **User/pass discrepancy** — enumeration via differing "bad user" vs "bad password" responses.
- **Session fixation** — session ID not rotated on login lets a planted ID be reused post-auth.
- **Two-persona testing** — low-priv + admin accounts replayed across each other's functions; automated by the **Autorize** Burp plugin.
- **IDOR** — insecure direct object reference: substitute UIDs/hashes/emails in requests; increment/decrement/negate.
- **Content-Type downgrade** — CSRF/bypass trick switching `application/json` → `text/plain` to skip preflighted checks.
- **File polyglot** — one file valid as two formats (GIF+JS, PDF+HTML); bypasses type checks, stores active content same-origin.
- **Blacklist bypass ladder** — ordered rotations for redirect/include filters: escaped slashes, scheme stripping, encoding, traversal mutations.
- **Transport gap** — HTTPS missing somewhere: insecure images, analytics carrying session data/PII, missing HSTS/`Secure` flags.
- **"Noise" vulns** — low-sev auxiliary classes: content spoofing, Referer leakage, missing headers, path disclosure, clickjacking.
- **n-minute assessment** — Haddix's ordered 9-step battery for maximum yield in a fixed time window (diminishing-returns first).

## Tools referenced

- **Nmap** — `nmap -sS -A -PN -p- --script=http-title`: SYN scan, service/OS fingerprint, no ping, all ports, HTTP titles.
- **Burp Suite** — proxy + scanner; plugins in TBHM: **Autorize** (authz diffing), **SQLiPy** (sqlmap bridge).
- **SQLMap** — automated SQLi; `-l` parses Burp logs, `-r` takes a saved request, tamper scripts for blacklists.
- **Liffy** — LFI testing tool; SecLists `JHADDIX_LFI.txt` = the paired fuzz list.
- **Wappalyzer / BuiltWith** — browser fingerprinting of platform/framework stack.
- **idb** — iOS app assessment tool (era; modern: MobSF/Objection/Frida — see `owasp-mas`).
