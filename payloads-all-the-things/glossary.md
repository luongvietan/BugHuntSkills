# Glossary — payloads-all-the-things

Terms used across the chapters. Cross-references point at chapter files.

## Payload mechanics

- **Payload family** — a group of payloads sharing a context/prerequisite and expected signal; the organizing unit of this skill.
- **Context** — where input lands in a parser (HTML body, quoted attribute, JS string, SQL clause, template tag, file path, header). Payloads are context-specific; misclassification = silent failure.
- **Detection payload** — inert probe proving evaluation (`{{7*7}}`, `SLEEP(5)`, `<!ENTITY x "y">`) before exploitation.
- **Polyglot** — a payload valid in several contexts simultaneously (XSS polyglots work in HTML/attr/JS; SQLi `SLEEP(1) /*' or ...` works in bare/quote/dquote).
- **Oracle** — the observable channel carrying the answer (response content, status, size, timing, errors, DNS/HTTP callback).
- **OOB (out-of-band)** — retrieving results through a side channel (DNS lookup, HTTP callback, UNC path) when the response carries nothing. Requires tester-controlled infrastructure.
- **Blind** — no direct output; inference via boolean pairs, timing, or OOB.
- **Second-order** — payload stored safely, executed later by a different code path/query.
- **Gadget / gadget chain** — existing code (class methods, magic methods, template utilities) chained so deserialization or template eval reaches a dangerous function. phpggc/ysoserial/ysoserial.net generate them.
- **CSTI vs SSTI** — client-side (AngularJS `{{ }}` in the browser) vs server-side (Jinja2/Twig on the server) template injection.

## Protocols & formats

- **DTD** — document type definition; the `<!DOCTYPE ...>` block where XML entities live (ch06).
- **Entity (general/parameter)** — `&name;` usable in XML content; `%name;` usable only inside the DTD — the key to blind/error XXE techniques.
- **XInclude** — `<xi:include>` file/URI inclusion inside XML documents; works when `<!DOCTYPE>` injection is impossible.
- **Wrapper / stream wrapper** — `php://`, `file://`, `data://`, `expect://`, `zip://`, `phar://`, `gopher://`, `dict://`, `jar://`, `netdoc://` — URI schemes that change how a path is read/fetched (ch05, ch07).
- **php://filter** — read-transform wrapper; `convert.base64-encode` dumps PHP source instead of executing it; chained iconv/base64 filters can synthesize code (filter-chain RCE).
- **PHAR** — PHP archive; `phar://` access unserializes its metadata — a deserialization gadget path (ch10).
- **JNDI** — Java naming/directory interface; `lookup()` on attacker URL → remote class loading (LDAP/RMI refs via marshalsec).
- **ViewState** — ASP.NET serialized page state (`/w...`, `FF01`); forgeable with `machineKey`/validation keys via ysoserial.net.
- **HPP** — HTTP parameter pollution: duplicate params; stacks resolve to first/last/all/array differently (ch14).
- **CRLF injection** — `%0d%0a` in header-bound input → split the HTTP response (cookie set, redirect, body spoof).
- **Smuggling / desync** — front/back disagreement on request boundaries: CL.TE, TE.CL, TE.TE, H2 downgrade, client-side desync (ch14).
- **Cache deception/poisoning** — suffix tricks to cache private pages under static URLs; unkeyed input tricks to store attacker content (ch14).
- **XS-Leak** — cross-site oracle (timing, frame count, errors, cache) leaking bits when CORS blocks reads.
- **CSWSH** — cross-site WebSocket hijacking: WS handshake honoring cookies without Origin checks.

## Injection classes

- **Tautology** — always-true clause (`' OR '1'='1`) for auth bypass; pair with `LIMIT 1` to bound the match set.
- **UNION injection** — `UNION SELECT` appends attacker rows; requires matching column count/types.
- **Error-based** — force data into a DB/parser error (`CAST(...AS int)`, `extractvalue`, `json('')` oracle).
- **Boolean blind** — true/false response differences (`AND 1=1` vs `AND 1=2`) → binary-search characters.
- **Time-based** — `SLEEP`/`WAITFOR`/`pg_sleep`/`BENCHMARK` delays as oracle.
- **Routed injection** — first query's output is concatenated into a second query; inner payload hex-encoded (`0x27...`).
- **NoSQL operator injection** — `[$ne]`/`[$gt]`/`[$regex]`/`$where` operators injected through urlencoded or JSON params (ch03).
- **LDAP filter injection** — `*)(`, `(|(`, `)(&` break filter structure; `userPassword:2.5.13.18` octet-match extraction.
- **SSTI breakout** — from `{{ }}` expression to engine's object graph (`__class__.__mro__`, `?new()`, `T(java.lang.Runtime)`) → code exec.
- **SSI/ESI** — `<!--#directive -->` (server includes) vs `<esi:...>` (edge-surrogate includes) — different processors, similar idea.
- **Argument injection** — input appended as a CLI argument; inject a flag (`-o`, `--output`, `-oProxyCommand`) rather than a command separator.
- **WorstFit** — Windows ANSI fullwidth-character quoting trick bypassing `escapeshellarg`-style wrappers.

## Identity & tokens

- **JWT** — `b64(header).b64(payload).b64(sig)`; attack surface: `alg` (none/HS↔RS confusion), `kid` (path/SQLi injection), `jku`/`jwk`/`x5u` (key injection), weak HMAC keys (offline crack), claim tampering.
- **JWKS/jku** — JSON Web Key Set URL; pointing it at your own JWKS = key-injection signing.
- **XSW** — XML Signature Wrapping: duplicate SAML assertions so the verifier signs off on the original while the app consumes a forged copy (XSW1–XSW8).
- **Signature stripping** — removing the signature block entirely; vulnerable SPs accept unsigned assertions.
- **OAuth `redirect_uri`** — code/token return address; open-redirect chaining and scheme tricks turn it into a token relay.
- **`state` (OAuth)** — CSRF binding for the callback; missing/unchecked `state` → forced account linking.
- **ATO** — account takeover; reset-poisoning, token leaks (Referer), IDOR on reset, username/unicode collisions.
- **Magic hash** — `0e...` strings that loose-compare as `0 == 0` in PHP (`240610708`, `QNKCDZO`).
- **Type juggling** — loose `==`/`!=` semantics producing true for mismatched types.
- **Mass assignment** — ORM binding all request fields → inject privileged properties (`isAdmin`, `role`).
- **BOLA/IDOR** — broken object-level authorization via direct identifier references.
- **Single-packet attack** — HTTP/2 multiplexed requests landing simultaneously; the standard race-condition delivery.

## Infra terms

- **DNS rebinding** — low-TTL answers that flip between IPs; bypasses resolve-then-fetch checks and same-origin assumptions.
- **nip.io / 1u.ms / localtest.me** — wildcard-DNS services mapping `<name>.<ip>` to that IP / alternating answers.
- **Vhost brute** — host-header fuzzing for internal sites sharing an IP.
- **Surrogate/ESI** — caching intermediaries (Varnish, Squid, Fastly, Akamai) evaluating `<esi:*>` tags in responses.
- **machineKey** — ASP.NET keys signing/encrypting ViewState; leaked → forged ViewState RCE.
- **secret_key_base** — Rails session-cookie signing key; leaked → forged + deserializable cookies.

## Channels & tooling terms

- **OAST/interactsh/Burp Collaborator** — callback infrastructure for blind detection (DNS/HTTP hits).
- **Turbo Intruder / gate** — Burp extension + mechanism to release queued requests simultaneously.
- **FFUF/Sniper/Pitchfork/Cluster bomb** — fuzzing tool & Intruder attack modes for position/payload-set mapping.
- **nuclei templates** — `http/default-logins`, `http/exposed-panels`, `http/exposures` for management-interface discovery.
- **EICAR** — standard AV-test string file; safe way to fingerprint upload scanning.
- **CAPEC/CWE anchors** — the source repo's reference vocabulary; CWE-79 (XSS), CWE-89 (SQLi), CWE-918 (SSRF), CWE-611 (XXE), CWE-502 (deserialization), CWE-352 (CSRF), CWE-918/441… (use for report taxonomy).
