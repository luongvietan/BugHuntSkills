# Basic Toolkit — What zseano Actually Uses

> **Era note:** the *stack shape* (proxy + subdomains + content discovery + screenshots) is durable; named tools are dated (httprobe-era). Version-verified equivalents live in `recon-pipeline`'s runbook.

## Core Idea
Minimal tool stack: one proxy, a recon chain, a fuzzer, good wordlists, and a few self-built scripts that all serve one purpose — **finding new content, parameters, and functionality before anyone else**.

## Tools & Commands

**Burp Suite** — the core proxy; Community edition is enough to start (Pro adds BApps + Collaborator; self-host collaborator if possible). BApp Store for extensions.

**Subdomain & host discovery chain:**
```bash
# Thorough subdomain enum (passive+active+alterations)
amass enum -brute -active -d domain.com -o amass-output.txt

# Probe live http(s) incl. extra ports
cat amass-output.txt | httprobe -p http:81 -p http:3000 -p https:3000 \
  -p http:3001 -p https:3001 -p http:8000 -p http:8080 -p https:8443 -c 50 \
  | tee online-domains.txt

# Diff new vs old domain lists
cat new-output.txt | anew old-output.txt | httprobe

# Permutation-based discovery (finds gems)
cat amass-output.txt | dnsgen - | httprobe

# Visual triage (accepts endpoints/files too, not just domains)
cat domains-endpoints.txt | aquatone
```

**Content discovery:** `ffuf -ac -v -u https://domain/FUZZ -w wordlist.txt` — fastest/most customizable; read the full docs.

**Wordlists:** SecLists for breadth; CommonSpeak to *generate* custom lists; own lists built from program keywords > generic lists — "don't blindly use wordlists; use meaningful ones."

## Custom Tools (the pattern matters more than the code)

- **WaybackMachine scanner** — scrape `/robots.txt` + homepage of every subdomain across all years; dead endpoints get re-fuzzed for still-alive status; old `.js` files leak forgotten code.
- **ParamScanner / InputScanner** — scrape each endpoint for `<input>` names/ids and `var {name} =` in JS; feed back as parameters. Alternatives: LinkFinder (JS URL extraction), parameth (param brute force).
- **AnyChanges** — watch URLs for new `<a href>` links and new `.js` references → catch unreleased features first.

**Tooling philosophy:** "Can you spot the trend? I'm trying to find new content, parameters and functionality to poke at." Old files from 7 years ago still on the server have yielded full account takeovers.

## Anti-patterns

- **Tool sprawl** — few tools mastered > many tools owned.
- **Tools over hands** — "I prefer seeing what's in front of me and understanding how it works"; recon serves manual hacking, not vice versa.
- **Generic wordlists only** — build per-target lists from discovered keywords (CommonSpeak, notes).

## Key Takeaways

1. Recon chain: `amass → httprobe (extra ports) → anew/dnsgen → aquatone`.
2. `robots.txt` per subdomain is the triage signal for "worth deeper scanning."
3. Wayback = time-machine recon; forgotten files = forgotten code paths = bugs.
4. Scrape parameters from inputs + JS vars; brute hidden params (`parameth`) for unreferenced functionality.
5. Monitor `.js` changes daily — code ships before features go live; flip `true`→`false` flags.

## Connects To

- **ch06**: where these tools run in the methodology (Step Two)
- **ch01**: custom wordlists come from notes
- **ch07**: what to automate vs. keep manual
