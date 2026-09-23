---
name: web-security-academy
description: PortSwigger Web Security Academy topic guides distilled into a hunting reference. Use when testing for modern web vuln classes no book covers — HTTP request smuggling (CL.TE/TE.CL/TE.TE, HTTP/2 desync H2.CL/H2.TE, response queue poisoning, request tunnelling, CL.0/client-side desync), web cache poisoning & deception, Host header attacks, prototype pollution (client & server-side), DOM clobbering & taint-flow sinks, WebSocket attacks (CSWSH), JWT attacks, OAuth 2.0/OIDC flaws — plus the standard classes (XSS, SQLi/NoSQLi, CSRF, CORS, clickjacking, SSRF, XXE, command injection, SSTI, path traversal, file upload, insecure deserialization, access control, authN, info disclosure, logic flaws, race conditions, API/GraphQL, LLM attacks). Each chapter runs mechanism → detection signals → exploitation → lab reference. Assume all testing is authorized and in scope.
---

# Web Security Academy — PortSwigger Topic Guides

Knowledge base distilled from PortSwigger's Web Security Academy (the free training platform behind Burp Suite). The Academy's signature is research-driven classes that originated in PortSwigger Research whitepapers — desync attacks, cache exploitation, client-side prototype pollution — and every topic ships with deliberately vulnerable labs.

**This skill is the set's authority for the modern classes.** When the orchestrator routes request smuggling, HTTP/2 desync, cache poisoning/deception, Host header abuse, prototype pollution, DOM clobbering, WebSocket attacks, JWT, or OAuth work, it lands here — no book-based skill covers them with this depth.

Mental model: the web is a *pipeline* — browser → CDN/reverse proxy → front-end → back-end — and every component boundary is a parsing disagreement waiting to be exploited. Desync = two servers disagreeing on request boundaries. Cache poisoning = cache and origin disagreeing on keys. Cache deception = cache and origin disagreeing on path meaning. Prototype pollution = parser and runtime disagreeing on what `__proto__` means. Find the boundary, exploit the disagreement.

Related skills: `bug-bounty-bootcamp` + `web-hacking-101` (same classic classes, book-driven loops), `bug-bounty-playbook` + `zseano-methodology` + `tbhm-methodology` (methodology), `owasp-wstg` (checklist authority), `hacking-apis` + `owasp-api-security-top-10` (API depth), `xss-cheat-sheet` + `payloads-all-the-things` (payload refs), `web-app-hackers-handbook` (legacy protocol coverage).

## How to use

**Modern classes — the reason this skill exists**
- HTTP request smuggling (CL.TE / TE.CL / TE.TE, detection, classic exploitation) → `chapters/ch01-request-smuggling.md`
- HTTP/2 & advanced desync (H2.CL/H2.TE, response queue poisoning, request tunnelling, CL.0/client-side desync) → `chapters/ch02-http2-desync.md`
- Cache poisoning, cache deception, Host header attacks → `chapters/ch03-cache-host.md`
- Prototype pollution (client + server) → `chapters/ch04-prototype-pollution.md`
- DOM taint-flow sinks & DOM clobbering → `chapters/ch05-dom-attacks.md`
- WebSocket vulnerabilities & CSWSH → `chapters/ch06-websockets.md`
- JWT attacks → `chapters/ch07-jwt.md`
- OAuth 2.0 & OpenID Connect → `chapters/ch08-oauth.md`

**Cross-origin client attacks**
- CORS + CSRF + clickjacking → `chapters/ch09-cors-csrf-clickjacking.md`
- XSS (reflected/stored/DOM, contexts, CSP) → `chapters/ch10-xss.md`

**Server-side classics**
- SQL injection + NoSQL injection → `chapters/ch11-sqli-nosql.md`
- SSRF + XXE → `chapters/ch12-ssrf-xxe.md`
- Command injection + SSTI + path traversal + file upload + deserialization → `chapters/ch13-injection-files.md`
- Access control + authentication + information disclosure → `chapters/ch14-access-authn-info.md`
- Business logic + race conditions + API testing + GraphQL + LLM → `chapters/ch15-logic-race-api-llm.md`

- Terms → `glossary.md`; cross-cutting heuristics → `patterns.md`; quick ref → `cheatsheet.md`
- Lab index (free, registration required): `https://portswigger.net/web-security/all-labs` — each chapter ends with its topic anchor.

## The Academy doctrine

1. **Every vulnerability is a disagreement.** Front-end vs back-end, cache vs origin, sanitizer vs browser, spec vs implementation. Map the components, then attack the seam.
2. **Timing is a detection oracle.** Desync probes, blind injection, race windows — when nothing reflects, delay deltas are the signal. Baseline first; a 5-second delta on a 200ms baseline is a finding.
3. **Ambiguity beats payload size.** A duplicated header, a double `Host`, an obfuscated `Transfer-Encoding`, an ambiguous path — the smallest malformed request often beats the cleverest payload.
4. **Chained impact is the point of smuggling/cache bugs.** A desync alone is a parser quirk; desync → response queue poisoning → stolen session → ATO is a report. Always finish the chain.
5. **Keyed vs unkeyed is the whole cache game.** If a header/cookie/param isn't in the cache key but reflects in the response, you have a poison primitive. Param Miner finds them mechanically.
6. **Source → sink → gadget.** Prototype pollution and DOM vulns need three parts; polluting a property nothing reads is trivia. Trace the taint, then prove the gadget.
7. **Read the front-end's rewrite.** Smuggled requests that echo back injected headers (`X-Forwarded-For`, session cookies in bodies) tell you exactly what the proxy adds — that knowledge is itself the exploit.
8. **Lab-verified technique transfers to bounty.** Academy labs are deliberately vulnerable; on real targets apply the same probes gently — desync and cache attacks can hit other users' traffic, so confirm with minimal requests and your own account.

## Scope & ethics

All testing is assumed authorized and in scope (bug bounty / pentest / Academy labs). Extra restraint for the classes here: request smuggling and cache poisoning affect *other users'* responses — prefer detection probes that only affect your own session (timing, cache-buster params, your own cached page) and stop at the minimal proof. WebSocket and LLM testing can generate real messages/actions in shared systems — use test accounts and test workspaces. Follow each program's rules on volumetric testing before racing or fuzzing.
