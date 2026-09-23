# Ch15: Exploiting Information Disclosure

Source: Chapter 15. Leaks are force multipliers: each disclosed detail sharpens the next attack — error text reveals code paths and SQL fragments, debug data hands you tokens and params, public sources map internals, and *inference* extracts data the app never displays.

## Sources

- **Error messages**: SQL/DB errors (query fragments → schema hints → tune injection), stack traces (file paths, framework versions, code structure), verbose debug output (runtime state, config, sometimes credentials/session tokens), server/DB connection errors revealing internal hosts and ports.
- **Debug artifacts**: debug parameters (`debug=1`, `test=`), commented-out HTML/JS, test functions, logging endpoints, error pages that echo request data (see ch10 logic ex.11 — static error stores leak *other users'* data).
- **Public/published**: Wayback/search caches, job postings (tech stack), newsgroup/forum posts by staff (internal naming, IP ranges, product versions), WSDL/service descriptors, `robots.txt`, sitemap, `.git`/`.svn`/`CVS` metadata dirs, backup files.
- **Client-side leak**: JS source (endpoint map, param names, commented creds, hardcoded keys), HTML comments, autocomplete caches, mobile/desktop app bundles.
- **Inferred data**: match-count oracles, timing differences, behavioral deltas — a channel that reveals 1 bit per request is still a channel.

## Weaponizing disclosure

- Stack trace → map code structure, find include paths for traversal, learn library versions → search known CVEs.
- SQL error → complete query shape → construct working UNION/blind payload without blind iteration.
- Internal IPs/hostnames → SSRF target list, DNS patterns, subnet guesses.
- User data in shared debug/error stores → token harvest → session hijack (ch10 ex.11).
- Version strings → default credentials/content for that product (ch17 app server).
- CSRF-token/session-data echo → cross-user fixation data.

## Eliciting disclosure deliberately

- Provoke errors on purpose: type confusion (string where int expected), malformed structures, overlong input, nulls, invalid syntax at every interpreter — each subsystem (web server, app framework, DB, XML parser, mailer) errors differently, and each error is a fingerprint.
- Force code paths: remove required params, submit out-of-sequence, hit error branches directly.
- Search public sources systematically: `site:target` + error strings, employee handles, product names + forum posts, cached admin/login pages.

## Defense signals to report

- Generic error handling everywhere (custom error pages, no stack traces), debug switches off in production, no sensitive data in errors/logs, minimal client-side commentary, predictable response shape across valid/invalid (no oracle).

## Checklist

- [ ] Every error condition provoked at every layer; message content catalogued.
- [ ] Public sources mined: archives, search, job posts, forums, metadata files.
- [ ] Client code reviewed: comments, endpoints, keys, debug flags.
- [ ] Each disclosure mapped to the attack it enables (chain accounting).
- [ ] Inference channels tested where direct disclosure fails (counts, timing, deltas).
