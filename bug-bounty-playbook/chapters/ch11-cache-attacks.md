# Ch 11 — Web Cache Poisoning + Deception

> **Shared-cache boundary:** poisoning a cache entry serves your payload to *other users*. Prove with your own cache-buster parameter / your own cached page (`web-security-academy` ch03 rules); a victim-facing poison entry is shared-infra impact requiring explicit permission.

## How caching works (the model that matters)

Cache sits in front: first request for a key → web server → response saved; later identical requests served from cache. "Identical" is decided by **cache keys** — usually only method + path + host (e.g. `GET /embed/v4.js?_=160…`, `Host: play.vidyard.com`). Everything else is **unkeyed input**: it changes the response but NOT the cache lookup. The `Vary` header lists extra keyed values (`Accept-Encoding`, `User-Agent`…). Cache indicators in responses: `X-Cache: hit|miss`, `Age: <seconds>`.

## Web cache poisoning — self-XSS → stored XSS

**Recipe:**
1. **Find unkeyed input** — Burp **Param Miner** plugin: right-click → Param Miner → OK (or "guess headers" / "guess GET parameters"). Results under Extender tab. Example find: `X-Forwarded-Scheme` — unkeyed AND reflected.
2. **Weaponize the unkeyed input** — does it reflect? Self-XSS, open redirect, header injection. Normally self-XSS is unpayoutable…
3. **Force the poisoned response into cache** — page already cached won't re-render. Path is a cache key → **append/change a random GET param** (`?test=2`) → `X-Cache: miss` + `Age: 0` = fresh cache write.
4. Re-send with payload in the unkeyed header + bumped param → payload is now **cached and served to every visitor** → self-XSS becomes stored XSS.
   *On a program you stop here:* your own cache-busted URL serving your
   marker is the PoC — "served to every visitor" is the report's impact
   paragraph, not a state you leave live. Purge/expire your entry after
   the screenshot.

**Checklist**: unkeyed input (param miner) → exploitable reflection → cacheable response (`miss`/`Age` behavior) → confirm by fetching URL in a fresh browser.

## Web cache deception — trick the cache into storing PII

Opposite direction: get the victim's *private* page cached publicly, then read it.

**Two ingredients:**
- **Path confusion**: app serves the same resource regardless of trailing path — `/users/me` ≡ `/users/me/nonexistent.css` (catch-all routing).
- **Static-extension caching**: cache rules cache `*.css`/`*.jpg`/`*.png` even when `Cache-Control: no-cache` says don't — the executive-decision override.

**Payloads** (from "Cached and Confused" paper):

```
/users/me/nonexistent.css            → path parameter
/users/me%0Anonexistent.css          → encoded newline (server stops at %0A, cache doesn't)
/users/me%3Bnonexistent.css          → encoded semicolon (server: param, cache: path)
/users/me%23nonexistent.css          → encoded # (server: fragment → ignores rest)
/users/me%3fname=valnonexistent.css  → encoded ? (server: query, cache: path)
```

Web server and cache **parse the same URL differently** → server returns the private page → cache stores it under a public `.css` URL.

**Recipe**: find page with sensitive per-user data (`/users/me`, settings, billing) → try path-confusion suffixes → check `X-Cache: miss→hit`/`Age` → fetch URL unauthenticated → victim PII for anyone with the link → potentially ATO (session/reset tokens on the page).
*Live-program bound:* run this against **your own** `/users/me` page — the deception works on your data, the shared-cache PII sentence is the demonstrated mechanism you describe, not other users' pages you fetch.

## Difference in one line

- **Poisoning** = attacker writes malicious content into a shared cached page (attack users at scale).
- **Deception** = victim's private page becomes publicly cached (steal their data).
