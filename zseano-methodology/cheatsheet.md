# Cheatsheet — zseano Methodology Decision Rules

> Era-marked tooling routes to `recon-pipeline`; per-class proof ceilings in chapter callouts and `payloads-all-the-things` (see `sources.md`).

## The loop
1. Pick wide-scope program (months horizon) → 2. Mine disclosed bugs for leads → 3. Manual first pass (feature question lists, hunt FILTERS) → 4. Dork + subdomain/robots/wayback + param replay → 5. Deep second pass (read every `.js`) → 6. Automate recon + diff monitoring → 7. Rotate 5-6 programs; patches & new features = new leads.

## "Where there's a filter, there's a bypass" — filter map

| Filter symptom | Likely gap to try |
|---|---|
| `<script>` → `&lt;script&gt;` but `%26lt;` → `<` | double-encode chain |
| blacklist of whole tags | `<svg>`, `</script/x>`, `<ScRipt>`, incomplete `<iframe src=` |
| checks complete tags only | unclosed tag — rest becomes attr |
| checks param VALUES | put payload in param NAME |
| referer==domain check | absent referer (meta no-referrer / data: iframe) or `theirdomain.com.evil.com` |
| redirect whitelist `*.target.com` | open redirect on target.com → token leak |
| SSRF blocks `127.0.0.1`/`localhost` input | your-domain 302 → internal; DNS tricks; open-redirect chain |
| upload ext check | `x.php/.jpg`, `x.html%0d%0a.jpg`, no ext, `.html`+`Content-Type:image/png`, magic bytes + payload |
| `canEdit:false`, `feature:false`, premium gate | flip/test server-side — "false is the new true" |
| GUID "protection" | hunt leaks (filenames, source, Google); try plain integers anyway |
| only checks documented method | GET→POST/PUT — unhandled methods error-leak secrets |

## First-look quick battery (per feature)

- Params used? in source/JS? GET+POST both? hidden/unreferenced params?
- `<> " '` + `%00 %09 %0a %0d` + unicode in every field — reflected where? encoded how?
- Mobile app/API path — same checks? (usually weaker)
- Redirect params (`returnUrl goto return_url back`) — on login/register always?
- Role levels — guest→moderator→admin calls; free→paid features
- OAuth/social flows — token leakage, cancel-still-issues-token, trusted profile fields
- Upload — matrix test (ext/name/content-type/magic/CSP-host)
- New↔old feature seams — new flow bypassing old verification? sandbox/test values?
- `.js` per endpoint — comments, hidden endpoints, unreleased flags
- Host-header trust on reset email — `Host: evil.com` → token leak?

## Recon order (Step Two)

1. Dorks (functionality keywords + file exts + GitHub secrets) while scans run
2. `robots.txt` per subdomain → triage for deeper work
3. Wayback → old files/endpoints → re-fuzz alive
4. ffuf with *stack-matched* wordlist (dorked extensions tell you which)
5. Param replay sweep + Grep-Match
6. `.js` deep read + daily diff monitors

## Golden rules

- Expect the site to be secure — then challenge it; "testing if it works as intended" IS the job.
- Blind-XSS every stored param — black-box means unknown sinks.
- Encode `?&#/\` when chaining through redirects; double-encode across 2 hops.
- One-time requests (first app launch) — watch them or lose them.
- Patches ≠ fixed: probe the fix, then sweep the flaw class site-wide.
- Notes from day one → custom wordlists → treasure map.
- Burned out on a feature? Note it, move on, come back fresh.
- `$_GET` vs `$_POST`: always test both.
