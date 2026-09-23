# Ch03: Web Cache Poisoning, Cache Deception & Host Header Attacks

Sources: `/web-security/web-cache-poisoning`, `/web-security/web-cache-deception`, `/web-security/host-header`. **GAP CLASS.** All three exploit disagreements between an intermediary (cache/proxy) and the origin over *what a request means* — keyed vs unkeyed components, static vs dynamic paths, trusted vs untrusted hosts.

## Web cache poisoning — mechanism

Caches decide "same request?" by a **cache key**: usually method + path + `Host` (sometimes selected headers). Everything else is **unkeyed** — ignored for matching but still processed by the origin. If an unkeyed input reflects into the response, you can poison the cache entry served to everyone who matches the key.

Attack loop: **identify unkeyed inputs → elicit a harmful response → get it cached.**

- **Finding unkeyed inputs**: **Param Miner** (Burp extension) guesses headers/params and diffs responses. Manual candidates: `X-Forwarded-Host`, `X-Forwarded-Scheme`, `X-Original-URL`, `X-Rewrite-URL`, `X-Forwarded-For`, cookies, `User-Agent`, extra query params.
- **Detecting cache hits**: `X-Cache: hit/miss`, `Age`, `Cache-Control`, `Via`, `CF-Cache-Status` (Cloudflare), `X-Served-By` (Fastly). No headers? Time it — cached responses return measurably faster.
- **Cache-busters**: add a unique param (`?cb=12345`) while developing the payload so each test fills a *fresh* key — then drop it to poison the real key.

### Cache design flaws

- **Unkeyed input → reflected gadget**: `X-Forwarded-Host: attacker.tld` reflected into a `<script src>` / redirect / link → stored XSS-equivalent served to all visitors of that URL.
- **Resource-import abuse**: response imports a JS file from a host derived from an unkeyed header → your poisoned page loads attacker JS for every victim.
- **Cookie-handling bugs**: cookies that reflect into the page but aren't keyed.
- **Multiple headers chained**: payload needs two unkeyed headers together.
- **`Vary` header abuse**: `Vary: User-Agent` means the cache keys per UA — poison per-UA, or use it to fingerprint a keyed dimension; a response that *reveals* its cache key components (`Cache-Control`, `Vary`) tells you what you must keep constant.
- **DOM-based gadgets**: unkeyed input reaches a DOM sink — chain with `ch05-dom-attacks.md`.

### Cache implementation/key flaws (2020 research)

- **Unkeyed port** — `Host: target.com:1337`; port not in key but reaches the app.
- **Unkeyed query string** — cache ignores the query; poison `/?q=<xss>` served for `/`.
- **Cache parameter cloaking** — cache ignores certain params (`utm_*`, or everything after `;`/specific delimiters) while the app reads them.
- **Normalized cache keys** — cache normalizes the path (`/../`, encoding, case) differently from the origin → same key, different content.
- **Cache key injection** — delimiter/encoding tricks let one request write to another URL's key.
- **Internal caches** — app-level caches (fragment/full-page) behind the CDN; poison persists even when CDN config looks clean.

## Web cache deception — mechanism

Inverse of poisoning: trick the cache into *storing a dynamic, private response* as if it were static, then fetch it as the attacker. Requires the victim to open your crafted URL while authenticated.

- **Cache rules exploited**: static extension rules (`.css`/`.js`), static directory rules (`/static/`, `/assets/`), file-name rules (`robots.txt`, `favicon.ico`).
- **Path mapping discrepancies**: origin resolves `/account/foo.css` to `/account` (path info), cache sees a static `.css` request → caches the account page.
- **Delimiter discrepancies**: `;`, `,`, `..`, `%00`, `#`-like characters that origin treats as path separators/ignores but cache treats literally — e.g. `/profile;.css` or `/profile%0a.css`.
- **Delimiter decoding discrepancies**: cache and origin decode encoded delimiters differently (`%2e`, `%2f`, double-encoding).
- **Static directory rules**: `/static/../account` — origin normalizes to `/account`, cache matches the `/static` prefix (or vice versa — test who normalizes).
- **Normalization direction matters**: if the *origin* normalizes, put the traversal in the path; if the *cache* normalizes, encode so cache sees a benign static path while origin traverses.
- **Detecting normalization**: request `/AAA/../BBB` and encoded variants; compare origin-served vs cache-served responses to learn who resolved what.
- **Execution**: pick a private GET/HEAD endpoint (state-changing requests aren't cached) → find the discrepancy → build the URL → victim visits (CSRF-like delivery, email/chat link) → attacker re-requests the URL and reads the cached private content. Confirm with a cache-buster first; never test on strangers' data.

## Host header attacks — mechanism

`Host` routes requests among virtual hosts / intermediaries; apps that trust it for URL generation, password resets, or auth decisions are exploitable. Probes:

- **Arbitrary Host** — replace `Host` with junk: if the app still works or reflects it, it trusts Host.
- **Flawed validation** — `Host: target.com.attacker.tld`, `attacker-target.com`, absolute-form `GET https://target.com/`, port tricks, duplicate `Host` headers (some stacks take first, some last, some reject — a *disagreement* again).
- **Ambiguous requests** — absolute + relative request lines, line wrapping, `Host` in different positions.
- **Override headers** — `X-Forwarded-Host`, `X-Host`, `X-Forwarded-Server`, `X-HTTP-Host-Override`, `Forwarded` — proxies may honor these over `Host`.

Exploitation primitives: **password reset poisoning** (reset link built from attacker Host → token lands on your domain), **cache poisoning** (unkeyed Host variants write poison), **classic server-side bugs via Host** (SQLi/SSTI in generated URLs), **auth bypass** (internal vhosts accept `localhost`), **virtual host brute-forcing** (fuzz Host on one IP for hidden apps — wildcard certs and same-IP hosting reveal candidates), **routing-based SSRF** (absolute-URI form: `GET http://internal/ HTTP/1.1` — the intermediary routes where the request line says), **connection state attacks**, **SSRF via malformed request line**.

## Lab reference

`https://portswigger.net/web-security/all-labs#web-cache-poisoning` · `#web-cache-deception` · `#http-host-header-attacks`
