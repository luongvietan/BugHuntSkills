# Ch 1 — Basic Hacking: Known Vulnerabilities

The oldest trick that still wins: target → tech stack → known CVEs → PoC → exploit. Most hunters skip this phase entirely — free wins for those who don't.

## The two cycles

**Cycle A — target-driven:**
1. Visit target → identify software + version.
2. Search for vulnerabilities affecting it.
3. Find PoC exploit code.
4. Run exploit → verify.

**Cycle B — exploit-driven (1-days):**
1. Watch threat feeds (ExploitDB, Twitter/X infosec) for newly dropped exploits.
2. Grab/find the PoC *fast* — time is the whole game.
3. Mass-scan all your known targets before they patch.

## Step 1 — Identifying technologies

- **Wappalyzer** (browser plugin; CLI version for scanning hundreds/thousands of hosts). Alternative: builtwith.com.
- **Fallback**: Wappalyzer works off regexes — if it's not in the DB it returns blank. Check the page footer for "Powered by X" banners, comments, headers, file extensions.

## Step 2 — Identifying vulnerabilities

- **Google**: `<TECHNOLOGY> <VERSION> vulnerabilities` / `exploits`. Dig past page 1 — gold hides in old blog posts.
- **ExploitDB / searchsploit**: `./searchsploit "name of technology"` — gives vuln + PoC in one step.
- **NVD** (nvd.nist.gov/vuln/search): list CVEs per technology. Newer CVE = better odds of unpatched. A CVE without a PoC is unusable to you.

## Step 3 — Finding the PoC

- **GitHub** — search the CVE id; highest hit rate. **Beware fake PoCs** — unvetted third-party code, some malicious. Read before running.
- **ExploitDB** — vetted-ish, second source.
- If neither has it, it probably doesn't exist publicly → write your own or move on.

## Step 4 — Exploitation

- Set up a vulnerable VM first so you know what success looks like.
- Run against target, compare output.

## Decision rules

- Most sites are patched → don't marry one target; exploit at scale across your asset inventory.
- Speed matters most for 1-days; depth matters most for obscure stacks.
