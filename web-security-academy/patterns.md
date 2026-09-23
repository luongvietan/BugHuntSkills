# Patterns — Reusable Heuristics from Web Security Academy

Cross-cutting heuristics that apply across the Academy's topics. Per-class detail lives in `chapters/`; see also `bug-bounty-bootcamp/patterns.md` for the book-driven loop.

## The master pattern: boundary disagreement

```
Two components parse the same input differently → find the seam → exploit the disagreement
```

- Desync: front-end vs back-end on request boundaries (CL/TE, h2→h1 translation).
- Cache poisoning: cache vs origin on what the key is.
- Cache deception: cache vs origin on what the path means.
- Host header: router vs app on which host is authoritative.
- Prototype pollution: JSON parser vs JS runtime on what `__proto__` is.
- CSWSH: HTTP-layer vs WS-layer on what authenticates the connection.
- JWT: verifier spec vs implementation on what "valid signature" means.
- SSRF filters: the validator's URL parse vs the fetcher's URL parse.

**When stuck, ask: "who else parses this input, and do they agree with the first parser?"**

## Detection heuristics

- **Baseline → probe → delta**: record normal status/length/time for the target request before attacking. Every Academy detection technique is a *delta against a baseline* — a 5s delay on a 200ms endpoint, a 500 on a 200, a different body hash.
- **Timing is the universal oracle**: desync hangs, blind SQLi sleeps, race windows, blind SSRF DNS — when nothing reflects, inject a delay and measure.
- **Ambiguity over payload**: duplicated headers, double Host, obfuscated TE, absolute+relative request lines, encoded delimiters — the minimal malformed request is the sharpest probe.
- **Two responses for one request**: the signature of response-queue desync.
- **Cache headers are the oracle**: `X-Cache`, `Age`, `Via`, `CF-Cache-Status`, `Cache-Control`, `Vary` — they tell you what cached, for how long, and what was keyed.
- **Error messages are fingerprints**: template-engine errors name the SSTI engine; SQL errors name the DBMS; XML parser errors confirm entity processing.
- **OOB for anything blind**: Collaborator/interactsh listener catches blind SSRF, XXE, command injection, stored XSS callbacks.
- **A reflected marker beats 1000 fuzz payloads**: `INJECT_MARKER_123` in each input → search where it surfaces (response, DOM, email, another user's view) → then pick the context-appropriate exploit.

## Exploitation heuristics

- **Finish the chain**: parser anomaly → controllable primitive → victim-impacting action → reportable impact. A desync alone is trivia; desync→queue-poison→session theft is critical.
- **Unkeyed input hunt is mechanical**: Param Miner on every cached page; manual shortlist = `X-Forwarded-*`, cookies, UA, extra params.
- **Source→sink→gadget**: PP and DOM bugs need all three — never report a source without a reachable gadget.
- **Keyed constraint + unkeyed payload**: for cache poisoning, keep the key components of the poisoned URL identical to what victims request; put the payload only in unkeyed parts.
- **The front-end tells you its secrets**: smuggled requests that reflect injected headers reveal the proxy's rewrite rules — use them to craft the real attack.
- **Headers are the modern input surface**: `Host`, `X-Forwarded-*`, `Origin`, `Referer`, `Cookie` — the Academy's newer classes all live in headers, not params.
- **HTTP method & content-type are free mutations**: same endpoint via GET/POST/PUT/DELETE and form/JSON/XML often hits different parsers and different authZ.
- **Aliased batching beats rate limits**: GraphQL aliases, WS message floods, h2 multiplexing — many logical requests inside one transport request.
- **Second-order everywhere**: stored payloads fire elsewhere — admin panels, log parsers, email renderers, template engines processing saved content.

## Scope & safety heuristics (unique to these classes)

- **Desync/cache attacks hit other users**: confirm with self-only probes (cache-buster keys, your own session, timing) before any payload that poisons shared state; the minimal confirmation is the finding.
- **Never leave a poisoned cache**: re-request the benign URL to refill the entry after PoC; document cache TTL.
- **Racing is volumetric**: check program rules on automated/high-rate testing before parallel sends; 10-20 requests is usually enough to prove.
- **WS and LLM actions are real**: a CSWSH PoC or prompt-injected function call may send actual messages — use test accounts and harmless actions.
- **Lab vs live**: Academy labs guarantee the bug exists; on live targets the same probe returning clean = move on, don't escalate force.

## Workflow patterns

- **Topic → lab → live**: practice the technique on the Academy lab for that class (every chapter links its labs), then apply to the bounty target — the probe sequence is identical, the tolerance is lower.
- **Encode everything twice**: URL→double-URL, hex, entities, unicode — traversal, filters, delimiters, and parsers each fold on a different encoding.
- **Read `/.well-known/` and machine-readable docs**: OAuth/OIDC discovery, `openapi.json`, `swagger`, `robots.txt`, `.map` files — free architecture maps before a single fuzz.
- **Diff per component**: run the same request over HTTP/1.1 vs HTTP/2, with/without each header, via different content-types — the delta isolates which component does what.
