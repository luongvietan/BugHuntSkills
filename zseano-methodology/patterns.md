# Patterns & Techniques — zseano's methodology

## Filter-First First Pass
**When to use**: first hours on any new program
**How**: probe features that *should* be protected; catalog what the filter blocks/allows (encodings, tags, extensions); treat each filter as both a bug-lead and a dev-mindset fingerprint
**Trade-offs**: slower than spraying, but findings are deeper and reusable across the app

## Question-Driven Feature Testing
**When to use**: every feature, every pass
**How**: fixed battery per feature — what params? where reflected? mobile same? roles? redirects? oauth? files? docs? (full lists in ch05)
**Trade-offs**: requires patience; prevents coverage gaps and spray-praying

## Encoding Reflection Ladder
**When to use**: any reflected input, any context
**How**: benign tag → entity/URL-encoded → double-encoded → `%00 %09 %0a %0d` variants → mixed/incomplete tags; each *differential* between encodings marks the bypass path
**Trade-offs**: systematic but needs per-context adaptation

## OAuth + Open-Redirect Chain
**When to use**: target has OAuth/social login whitelisting `*.target.com`
**How**: find open redirect *on* target.com → set it as `redirect_url` → user logs in → token rides the redirect to your host; wiki/API docs show how to use the token
**Trade-offs**: redirect params may be dropped mid-chain — URL-encode `?&#` (sometimes twice)

## SSRF via Redirect
**When to use**: URL-fetching feature filters internal hosts
**How**: host a 302 script (XAMPP+ngrok); feed your URL → server follows to internal; try `sleep()` pre-redirect for timeout bugs; chain open-redirects when externals blocked
**Trade-offs**: blind SSRF needs OOB/differential evidence; respect scope

## Upload Filter Matrix
**When to use**: any file-upload feature
**How**: probe axes independently — extension (`.txt/.svg/.xml`), filename parsing (`x.php/.jpg`, `x.html%0d%0a.jpg`, no-name), content-type trust, magic bytes (`‰PNG`+payload), reflection of filename (XSS), storage domain (CSP coverage)
**Trade-offs**: malformed combos crash parsers — that's also signal

## GUID/IDOR Hunting
**When to use**: object refs in requests, esp. mobile APIs
**How**: numeric swap first (even against GUID-shaped IDs); hunt GUID leaks (public pages, filenames, Google); inject `id` into JSON/PUT; then pivot to role/feature boundaries
**Trade-offs**: true random GUIDs need a leak — search before brute-forcing

## Disclosed-Bug Lead Mining
**When to use**: before first request to the app
**How**: `domain.com vulnerability` + hacktivity + OpenBugBounty → learn what was found/fixed/how; test whether the fix is complete or bypassable
**Trade-offs**: heavily-hunted leads; value is the *pattern*, not the same bug

## robots.txt × Wayback Triage
**When to use**: after subdomain enumeration
**How**: fetch `/robots.txt` per host (disallowed = interesting); Wayback the same paths + homepage; resurrect old endpoints/files; re-fuzz them alive
**Trade-offs**: noise on huge scopes — prioritize interactive-looking hosts

## Param Replay Sweep
**When to use**: after collecting endpoint+param lists
**How**: Burp Intruder `/endpoint?p1=xss"&p2=xss"` across all endpoints; Grep-Match responses; test GET **and** POST
**Trade-offs**: shallow but fast coverage multiplier on wide scopes

## Change-Diff Monitoring
**When to use**: continuous, on chosen programs
**How**: daily diffs on pages/`.js` files/subdomains; flag new links, new endpoints, `feature:false` flags → test pre-release or enable manually
**Trade-offs**: build/maintenance cost — start with the 2-3 highest-value hosts

## Patch Intelligence
**When to use**: after every fix lands
**How**: read the patch behaviorally (new param? new check? method-bound?) → probe the fix itself (GET→POST leaked the "hash") → sweep same flaw site-wide
**Trade-offs**: some programs patch only reported endpoints — verify or exploit responsibly
