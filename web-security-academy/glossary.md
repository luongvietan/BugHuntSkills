# Glossary — Web Security Academy

PortSwigger-specific vocabulary and protocol terms. For generic bug-bounty terms see `bug-bounty-bootcamp/glossary.md`; for API terms see `hacking-apis`/`owasp-api-security-top-10`.

## Desync & pipeline

- **HTTP request smuggling / desync** — front-end and back-end disagree about where one request ends; leftover bytes prepend the next request.
- **CL.TE / TE.CL / TE.TE** — which length header each server honors: front-end-CL+back-end-TE, front-end-TE+back-end-CL, or both-TE with one tricked into ignoring it via obfuscation.
- **TE obfuscation** — malformed `Transfer-Encoding` forms (`xchunked`, space/tab tricks, duplicate headers) that one server accepts and another ignores.
- **H2.CL / H2.TE** — HTTP/2 front-end downgrading to HTTP/1 recreates CL/TE confusion inside HTTP/2-looking traffic.
- **HTTP/2 downgrade** — front-end speaks h2 to the client, h1 to the back-end, regenerating headers — the seam where most modern smuggling lives.
- **Pseudo-header injection** — abusing `:path`/`:authority`/header-value CRLF translation in h2→h1 downgrades to forge HTTP/1 structures.
- **Response queue poisoning** — smuggled *complete* request generates a second response that gets served to the next requester on the connection.
- **HTTP/2 request splitting** — injected boundary makes the downgrade emit two HTTP/1 requests from one h2 request.
- **Request tunnelling** — smuggling a second request *inside* an allowed first request to reach back-end resources with front-end privileges.
- **0.CL** — back-end ignores body length (treats CL as 0), parsing the "body" as the next request.
- **CL.0 / H2.0** — client-side desync where the *server* treats the browser's request as having no body; leftover poisons the victim's next request — deliverable from a web page.
- **Client-side desync** — desync triggered and exploited entirely from a victim's browser; includes pause-based variants that hold the boundary open with delayed responses.
- **Single-packet attack** — Burp's parallel-send that packs multiple requests into one TCP packet (h2) so they land in the same millisecond window — the modern race-condition trigger.

## Cache & host

- **Cache key / unkeyed input** — the request components a cache uses to match entries (usually method+path+Host); everything else is unkeyed — free payload space.
- **Cache buster** — unique param added while developing a poison payload so tests don't pollute the real cache entry.
- **Web cache poisoning** — poison a cached response via unkeyed input so all key-matching visitors get your payload.
- **Web cache deception** — trick the cache into storing *dynamic/private* content under a static-looking URL, then read the victim's cached page.
- **Path mapping / delimiter / normalization discrepancy** — origin vs cache disagree on what `/x/../y`, `/a;b`, or encoded chars resolve to — the deception primitive.
- **Host header attack** — abusing trusted `Host`/`X-Forwarded-Host` for reset poisoning, cache poisoning, vhost discovery, or routing-based SSRF.
- **Virtual host brute-forcing** — fuzzing `Host` on one IP to find hidden applications sharing it.
- **Routing-based SSRF** — absolute-form request line (`GET http://internal/`) that makes an intermediary route to an attacker-chosen host.

## Prototype pollution & DOM

- **Prototype pollution (PP)** — injecting properties into `Object.prototype` (or another built-in) via recursive merge of `__proto__`/`constructor.prototype` keys.
- **PP source / sink / gadget** — the three required parts: a poisoning input, a dangerous consumer, and the property connecting them.
- **DOM Invader** — Burp-browser tool that finds PP sources/gadgets and DOM XSS sinks automatically.
- **DOM clobbering** — injecting `id`/`name` elements so named DOM nodes overwrite globals (`window.x`) or properties (`form.attributes`) the JS trusts.
- **Taint flow** — source→sink tracing for DOM vulns; every sink family (eval, innerHTML, location, postMessage…) = its own vuln class.
- **Web message vulnerabilities** — `postMessage` handlers that skip `event.origin` validation accept attacker-window data into sinks.

## WebSockets

- **WebSocket handshake** — `Upgrade: websocket` + `Sec-WebSocket-Key` → `101 Switching Protocols`; the only HTTP-shaped part of a WS connection.
- **CSWSH (cross-site WebSocket hijacking)** — server authenticates the handshake by cookies alone without `Origin` validation → attacker page opens an authenticated socket through the victim's browser.
- **`wss://` / `ws://`** — TLS vs cleartext socket; `ws://` also enables MITM tampering.

## Tokens & OAuth

- **JWT / JWS / JWE** — token format umbrella; JWS = signed (what "JWT" usually means), JWE = encrypted.
- **`alg:none`** — unsigned-JWT algorithm value some verifiers accept.
- **Algorithm confusion** — RS256→HS256 swap: verify-with-public-key-as-HMAC-secret; the public key is public.
- **`jwk` / `jku` / `kid`** — JWT header params: embedded key, key URL, key ID — each an injection/fetch vector when the verifier trusts them.
- **Grant types** — OAuth flows: authorization code (server-side exchange) vs implicit (browser-delivered token, weaker).
- **`state` / `nonce`** — OAuth/OIDC CSRF-binding params; missing/predictable = login CSRF or token substitution.
- **`redirect_uri` validation** — the check that decides where codes/tokens get sent; loose parsing leaks them to attacker hosts.
- **OIDC / ID token / UserInfo** — OpenID Connect: standardized login layer on OAuth; ID token = JWT of identity claims.
- **Dynamic client registration / `request_uri`** — OIDC endpoints that accept client metadata or fetch the auth request by URL → SSRF/metadata injection surface.

## Client-side & cross-origin

- **SOP** — same-origin policy: send cross-origin freely, read only same-origin or CORS-permitted responses.
- **ACAO / ACAC** — `Access-Control-Allow-Origin` / `-Allow-Credentials`; reflected origin + credentials = readable authenticated cross-origin data.
- **`null` origin** — sandboxed/`data:` contexts produce `Origin: null`; whitelisting it + credentials = exploitable.
- **Preflight** — `OPTIONS` request browsers send before non-simple CORS requests.
- **SameSite** — cookie scope: `Strict` (same-site only), `Lax` (+top-level GET nav), `None` (cross-site, needs `Secure`); bypasses: Lax-GET actions, 2-min Chrome window, on-site redirect gadgets.
- **Clickjacking / UI redress** — transparent iframe over a sensitive control; `frame-ancestors`/`X-Frame-Options` are the defenses.
- **Frame-busting** — JS `top!=self` navigation; defeated by `sandbox` attrs and double-framing.
- **Dangling markup** — unclosed-tag injection that bleeds subsequent page content (CSRF tokens) into an attacker URL without running script.
- **CSTI** — client-side template injection (AngularJS-era `{{}}` sandbox escapes).

## Injection & server-side

- **Blind injection channels** — boolean/content delta, error message, time delay, OOB (Collaborator/interactsh) — pick by what the response offers.
- **Second-order injection** — payload stored benignly, executed in a later query/request context.
- **NoSQL operator injection** — `{"$ne":""}`/`{"$regex":"^a"}`/`param[$gt]=` semantics injected into Mongo-style queries; syntax injection = the classic `'` style.
- **SSTI detect→identify→exploit** — `{{7*7}}` probe → engine fingerprint (`{{7*'7'}}`: Twig 49 / Jinja2 7777777) → engine-specific sandbox escape.
- **Polyglot / config-override upload** — files valid as images AND code, or `.htaccess`/`user.ini` uploads that redefine an extension as executable.
- **SSPP (server-side parameter pollution)** — server builds an internal request from your params; dupes/truncation/REST-path tricks inject into that second request.
- **Mass assignment** — binding extra JSON/form properties (`role`, `isAdmin`) the object supports but the UI never sends.
- **GraphQL introspection / aliases / `__typename` probing** — schema self-description; alias-batched requests (rate-limit bypass); error-suggestion field discovery when introspection is off.
- **Prompt injection (direct/indirect)** — overriding an LLM's instructions in its input (direct) or in data it later reads (indirect); output handling and plugin permissions turn it into XSS/SSRF/actions.

## Academy infrastructure

- **Academy lab** — deliberately vulnerable app per topic at `portswigger.net/web-security/all-labs` (free account).
- **PortSwigger Research whitepapers** — the source papers: "HTTP desync attacks", "HTTP/2: The sequel is always worse", "Browser-powered desync attacks", "Practical Web Cache Poisoning", "Web Cache Entanglement", "Gotta Cache 'em all", "Hidden OAuth Attack Vectors", "Exploiting CORS misconfigurations".
