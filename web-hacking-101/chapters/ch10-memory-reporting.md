# Ch 10 — Memory Vulnerabilities + Report Writing

## Memory bugs (know they exist; hunt them later)

Book's honest advice: *skip these as a beginner* — high skill floor. But recognize the terrain because languages you touch daily are C underneath (PHP, Python runtimes).

- **Buffer overflow**: write past allocated space → overwrite data/code → crash or exec. Ice-tray model: allocated 10, wrote 11.
- **Read out-of-bounds**: same over-read direction. **Heartbleed** = client sends small heartbeat message + large claimed length → server reads & returns heap beyond the buffer → private keys, session data (17% of TLS web servers, 2014).
- **Null byte injection**: `%00`/`0x00` terminates C strings early — `thisis%00mystring` has length 15 but reads 6. Relevant when web apps call C libraries/FFI (file paths, parsers).
- **Memory corruption patterns**: `memcpy`/`strcpy` with a **fixed-size dest and variable-size source** (Python Hotshot `memcpy(self->buffer+idx, s, len)`); `strdup` assuming null-termination (libcurl `curl_easy_duphandle` OOB); null-prefixed filenames → array underflow (`phar_parse_tarfile`).
- **Audit heuristic**: in C/C++/native extensions, find data copied between memory locations with mismatched length assumptions. Grep source for `memcpy`, `strcpy`, `strncpy`, `sprintf`.

## Vulnerability Reports — the craft chapter

### 1. Read the disclosure guidelines FIRST

Author's own scar: first Shopify find was a known/asked-not-to-submit bug → closed, -5 rep. Scope, known issues, exclusions — read before testing and again before writing.

### 2. Include details, then more

Minimum (what Yahoo/Twitter/Dropbox ask):
- URL + affected params
- Browser/OS/app version
- **Perceived impact** — what could this bug *do to them*
- Repro steps

Beyond: screenshot or video PoC. Frame impact relative to their product (stored XSS on Twitter ≠ on a low-interaction site; privacy leak on PornHub > Twitter).

### 3. Confirm the vuln before submitting

- "Missing CSRF header" — did the params already carry an unguessable per-user token?
- Don't burn rep on maybes. *"Take the extra minute and confirm."*

### 4. Respect the company (the triager's world, per Adam Bacchus)

Triagers fight: **noise** (invalid reports cost money), **prioritization**, **confirmation** (vague reports without repro waste cycles — a video alone doesn't cut it), **resourcing** (often one part-time person), **fix time** (dev lifecycle is slow), **relationship management**, **press risk**. Report = make their job easy.

- New programs get flooded — give them room. Polite ping after ~2 weeks; escalate via platform support only after.
- Bounty disputes: "have a discussion why you believe it deserves a higher reward. Avoid asking for more without elaborating" (Jobert Abma).

### 5. Don't shout hello before crossing the pond

Mathias Karlsson's SOP-bypass story: Firefox accepted `http://example.com..` (malformed host) → Flash SOP bypass → 7% of Alexa top-10k exploitable incl. Yahoo. Wrote it up — then kept verifying: coworker's VM confirmed, updated Firefox… **bug already patched**. Lesson: confirm on fresh versions before announcing/submitting; certainty > speed.

## Author's stack (Tools chapter)

Burp Suite (Pro) · ZAP · KnockPy + enumall (subdomains) · EyeWitness (screenshots) · nmap · Wappalyzer · aws-cli (bucket tests) · GitRob (repo secrets) · MobSF/JD-GUI (mobile) · Recon-ng · SecLists · IPV4info.com · Google dorks.
