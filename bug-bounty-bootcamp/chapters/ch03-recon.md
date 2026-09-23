# Ch5: Introduction to Reconnaissance

> **Currency note:** the recon *stages* are durable; specific tools/flags have moved — current command runbook with version-verified syntax + allowlist-derived scope lives in `recon-pipeline`.

Source: Chapter 5. Recon = discovering the target's attack surface *before* testing: assets, technologies, and forgotten entry points. Li's split: **passive** (no packets to target: OSINT, cert logs, Wayback) vs **active** (talking to the target: brute-forcing, port scans — confirm the program allows it).

## The recon pipeline

1. **Manual walkthrough** — use the app end-to-end as a user, at every privilege level, with Burp recording. This is the highest-value "recon" there is.
2. **Tech fingerprinting** — response headers, cookie names, JS frameworks, `BuiltWith`/`Wappalyzer`, error pages, job postings + employee profiles (reveals stack).
3. **WHOIS / IP ownership / ASN** — `whois`, ASN lookups reveal netblocks the org owns → expand scope within policy.
4. **Certificate transparency** — `crt.sh`, Cert Spotter, Censys: parse SANs for subdomains (incl. internal-looking names).
5. **Subdomain enumeration** — passive (Sublist3r, Amass passive, cert logs) then active (`gobuster dns -d target.com -w wordlist`, SubBrute, Amass active, DNS zone-transfer attempt, Altdns permutations). Merge + dedupe: `sort -u w1.txt w2.txt`.
6. **Port & service discovery** — Nmap (deep, slow) / Masscan (fast, broad) on owned netblocks; passive via Shodan, Censys, Project Sonar. Look for admin panels, dev ports, forgotten services.
7. **Directory & file brute-force** — Dirsearch / Gobuster with SecLists + Commonspeak2 wordlists matched to the stack. Find admin panels, config files, backups, old endpoints.
8. **Screenshot review** — EyeWitness/Snapper across live hosts; skim for login portals, debug pages, default installs.
9. **S3 / cloud buckets** — permutations of `company`, `company-dev`, `company-backup` on s3.amazonaws.com + Grayhat Warfare. Test read first; test write only with a harmless file you then delete: `aws s3 cp TEST s3://BUCKET/ && aws s3 rm s3://BUCKET/TEST`.
10. **GitHub & code leaks** — search org repos, issues, commits, history/blame for hardcoded secrets, internal endpoints, config files, outdated deps. Tools: Gitrob, truffleHog, gitleaks, PasteHunter, Wayback Machine.
11. **Google dorking** — `site:target.com inurl:admin`, `ext:log|sql|conf`, filetype searches via Google Hacking Database.

## Automating recon with bash

Chain the pipeline into `recon.sh` — `chmod +x` (700 for private), parameterize the target, redirect each tool's output to per-tool files, `sort -u` everything. Schedule with `cron`/`crontab` for **continuous recon**: diff each run against the last — newly appearing subdomains/endpoints are the least-hunted surface.

Command hygiene learned here: output redirection (`>`, `>>`), `grep` for filtering, `sort -u` for dedupe, command substitution, `chmod` for script perms.

## What to do with recon output

- Feed live hosts → Burp scope, directory brute-force, screenshot triage.
- Feed tech list → vuln-class checklist (WordPress → wpscan; GraphQL → introspection; old API versions → versioned-endpoint testing).
- Feed secrets/endpoints → code review (Ch22) and access-control testing.
- **Keep notes structured**: host, tech, interesting endpoints, auth model, ideas to try. Recon compounds across sessions.
