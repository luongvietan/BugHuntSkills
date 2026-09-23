# Ch02: HTTP/2 Desync & Advanced Smuggling

Source: `/web-security/request-smuggling` advanced section. **GAP CLASS — the deepest no-book coverage in the set.** HTTP/2 front-ends talking to HTTP/1 back-ends ("HTTP/2 downgrading") recreate the smuggling problem with new primitives — and HTTP/2-exclusive vectors smuggle without length confusion at all.

> **Lab vs live (bounty-safe):** same boundary as ch01 — H2 desync primitives
> are proved with differential/timeout probes; queue poisoning, response
> smuggling to other users, and request tunnelling that surfaces foreign data
> are shared-infra impact requiring explicit permission. Confirm the desync,
> write up the mechanism, stop.

## Mechanism

HTTP/2 replaces CL/TE with a built-in binary frame length — no ambiguity on the front-end hop. But when the front-end *downgrades* the request to HTTP/1 for the back-end, it must generate the HTTP/1 headers — and whatever it writes (`Content-Length`, `Transfer-Encoding`) reintroduces the disagreement:

- **H2.CL** — front-end emits HTTP/2 request with a stale/ignored `Content-Length`; downgraded back-end disagrees on the boundary → classic smuggle over an HTTP/2 entry point.
- **H2.TE** — HTTP/2 doesn't use TE, but a front-end that forwards `Transfer-Encoding` into the downgrade creates TE.CL-equivalent desync.
- **HTTP/2-exclusive vectors** — HTTP/2 forbids CRLF in header values, but some front-ends translate `\r\n` inside pseudo-headers or header values into raw CRLF in the HTTP/1 rewrite:
  - **CRLF injection via header values/names** — inject `\r\n` in an HTTP/2 header to forge new HTTP/1 headers (including a fake `Transfer-Encoding`/`Content-Length` split).
  - **Pseudo-header injection** — `:path`/`:authority` manipulation: supply an ambiguous host (`:authority` vs `Host` mismatch), ambiguous path, inject a full request line or URL prefix, or inject newlines the translation passes through.
  - **Hidden HTTP/2 support** — servers that only expose HTTP/2 on some routes/ALPN; probe `h2` support explicitly before assuming HTTP/1-only.
- **Response queue poisoning** — smuggle a *complete* request so the back-end generates two responses for one front-end request; the second response sits in the queue and is served to the next requester. Do it repeatedly and you harvest other users' responses (cookies, tokens) — the queue desyncs and the front-end may eventually break the connection, so confirm carefully.
- **HTTP/2 request splitting** — inject a boundary inside the `:path`/headers so the front-end's downgrade emits two HTTP/1 requests; account for front-end rewriting (it may fix your Host/authority).
- **HTTP request tunnelling** — when the front-end only allows certain requests, tunnel a second request inside a first (e.g. via HTTP/2 `CONNECT` or via a smuggled complete request) to reach the back-end with the front-end's privileges; leak internal headers via HEAD-based probes; blind tunnelling when nothing reflects.
- **0.CL request smuggling** — the *front-end* treats the request as having no body (interprets `Content-Length: 0` or implicitly zero) while the back-end honors the stated CL → the bytes the front-end forwarded anyway (the "body" it never expected) are prepended to the *next* request on the connection. Historically considered unexploitable; the 2025 browser-powered desync research showed client-triggerable paths make it real.
- **CL.0 / browser-powered smuggling** — flip it: the *client's* browser sends a request a server treats as CL.0; the leftover gets attached to the victim's next request on the same connection. Deliverable from a malicious web page — no Burp needed for the victim side. **H2.0** is the HTTP/2 analogue.
- **Client-side desync** — probe for desync reachable from a browser alone: trigger a desync that makes the server return a poisoned/redirected response for the victim's navigation, then import it as a script/CSS resource (client-side cache poisoning) or pivot into internal infra. **Pause-based desync** uses delayed responses (server- or client-side) to hold the boundary open.
- **Connection reuse (HTTP/1.1 keep-alive)** matters: browsers pool and reuse connections, so the smuggled remainder reliably lands on the *same user's or next user's* request.

## Detection signals

- In Repeater: switch protocol to HTTP/2, disable auto-normalization, and try **invalid/ambiguous pseudo-headers** and header values containing `\r\n` — look for 4xx/timeout deltas vs baseline.
- **Two responses for one request** = response queue poisoning candidate.
- Frontend-only differences: same request over HTTP/1.1 vs HTTP/2 produces different status/redirect/rewrite behavior → downgrade in play.
- Client-side vectors: an innocuous-looking response (200) on an ambiguous request, then a *different* response when the same URL is re-requested — the queue shifted.

## Exploitation

Escalation primitives, cheapest to deepest:

- **Response queue poisoning → session theft**: smuggle a *complete* request; the back-end emits two responses for one front-end request and the surplus response is served to the next requester. Repeat to harvest other users' responses — cookies, CSRF tokens, session-bound data. The queue eventually desyncs enough to break the connection, so pace probes and confirm with your own traffic first.
- **Request tunnelling → internal reach**: when the front-end only permits certain requests, tunnel a second request inside an allowed one (h2 `CONNECT`-style or a smuggled complete request) to hit back-end-only endpoints with front-end privileges; leak internal headers via `HEAD`-based probes; blind tunnelling when nothing reflects — confirm via timing deltas.
- **Client-side desync → resource-load hijack**: deliverable from a web page — desync the victim's connection so their next fetch (a JS/CSS import, a navigation) receives your poisoned or redirected response; chain into client-side cache poisoning or a pivot into internal infra the victim's browser can reach.
- **Pseudo-header/CRLF injection → front-end filter bypass**: h2 vectors slip past front-ends that only inspect valid HTTP/1 — smuggle paths/headers the WAF never sees.
- **Always finish the chain for the report**: parser anomaly → controllable primitive → victim-impacting action (session theft, stored response poisoning, internal access) → ATO on shared infrastructure.

## Lab reference

`https://portswigger.net/web-security/all-labs#http-request-smuggling` — advanced labs: H2.CL, H2.TE, CRLF injection, response queue poisoning, request tunnelling, CL.0, client-side desync. See `ch01-request-smuggling.md` for the classic set.
