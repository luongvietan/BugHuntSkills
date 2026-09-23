# Step Two — Expanding the Attack Surface

> **Volume + scope note:** expanding surface via brute-force/subdomain/content discovery is volumetric — allowlist-derived targets and policy rate limits apply (`recon-pipeline` class labels P/T/I). Discovered assets are leads until confirmed in scope.

## Core Idea
After the manual first pass, widen out: dork for indexed content, triage subdomains by *functionality* (not just liveness), mine `robots.txt` + Wayback for forgotten files, fuzz for content with meaningful wordlists, then do a **deeper second pass** through the main app — this time reading the `.js` files.

## Dorking (while scans run)

- **Functionality keywords** — prioritize interactive domains: `login, register, upload, contact, feedback, join, signup, profile, user, comment, api, developer, affiliate, careers, mobile, upgrade, passwordreset`.
- **File-extension dorks** — `php aspx jsp txt xml bak` reveal stack → pick the right wordlist; may surface sensitive files.
- **Google tricks people miss**: last page → "repeat the search with the omitted results included" doubles output; `-keyword` to exclude noise; re-run with **mobile user-agent** (different index!).
- **Beyond Google**: GitHub/GitLab dorks — `"domain.com" api_secret, api_key, apiKey, apiSecret, password, admin_password`; Shodan/BinaryEdge for exposed services; PublicWWW for code snippets; yougetsignal for co-hosted sites.

## Subdomain Triage (functionality over count)

- Scan `robots.txt` on every subdomain (Burp Intruder on `/robots.txt` position, or XAMPP-hosted redirect PHP to batch) — disallowed paths reveal third-party software and hidden content; it decides whether a subdomain deserves deep fuzzing.
- Prioritize by smell: `dev`, `prod`, `qa` keywords; third-party hosted (`careers.target.com`); anything with forms/APIs.
- **Grep-Match across all subdomain index pages** for keywords (`login`) — points effort at the right hosts fast.
- **WaybackMachine expansion**: old `robots.txt` entries + old homepage references — files from years ago still live; "high success with wide-scope programs and WayBackMachine."

## Content & Parameter Discovery

- Fuzz common endpoints first (`/admin`, `/server-status`) with the *right* wordlist for the revealed stack — expand based on hits; never spray generic lists blindly.
- **Parameter mining**: scrape `<input>` names/ids + JS `var` names per endpoint (InputScanner); then Burp Intruder replays common params across all discovered endpoints: `/endpoint?param1=xss"&param2=xss"` + Grep-Match on responses.
- **Test GET *and* POST** — bugs hide behind method differences (`$_GET` vs `$_POST` handling).

## The Deeper Second Pass (the real unlock)

- Go through *everything again* — first pass learned features, second reads the **per-endpoint `.js` files**: endpoint-specific code, developer comments, more endpoints, unreleased features (commented code with vulnerable params → report *before* public release).
- Build a daily `.js` diff monitor → test features pre-release.
- Why last: too much info early = overload/burnout; understand the app first, then how it's assembled.

## Anti-patterns

- **Subdomain collecting without triage** — thousands of hosts ≠ attack surface; functionality matters.
- **Trusting first-pass coverage** — "You can never find anything on your first look. You will miss stuff." (50+ passes on some assets.)
- **Ignoring omitted Google results** and desktop-only user-agent.
- **Static wordlists** — continuously grow custom lists from findings.

## Key Takeaways

1. Recon output → triage by interactivity: forms, logins, uploads, APIs, dev/staging, third-party.
2. `robots.txt` × every host × Wayback history = highest-signal cheap recon.
3. `.js` files per endpoint are the second-pass goldmine — comments, endpoints, unreleased flags.
4. Param replay across endpoints (`?p1=xss&p2=xss`) + Grep-Match = wide-scope bug multiplier; cover GET+POST.
5. Diff monitoring (.js, pages, subdomains) makes you first on new features — "code is here but feature isn't enabled" → can you enable it?

## Connects To

- **ch02**: the actual commands/tools used here
- **ch01**: notes/wordlists feed this expansion
- **ch07**: Wayback→ATO and `.js`→pre-release IDOR case studies
