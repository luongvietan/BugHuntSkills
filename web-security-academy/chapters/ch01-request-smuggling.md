# Ch01: HTTP Request Smuggling — Classic HTTP/1 Desync

Source: `/web-security/request-smuggling`. **GAP CLASS — no book covers this.** Request smuggling = interfering with how a site processes *sequences* of HTTP requests on a shared front-end→back-end connection. Front-end and back-end disagree about where one request ends and the next begins; the leftover bytes get prepended to the *next user's* request.

## Mechanism

HTTP/1 offers two ways to mark a request's end: `Content-Length` (byte count) and `Transfer-Encoding: chunked` (hex-length-prefixed chunks, `0\r\n\r\n` terminator). When both are present and the two servers pick different ones, the request desynchronizes:

- **CL.TE** — front-end honors `Content-Length`, back-end honors `Transfer-Encoding`. Front-end forwards the whole body; back-end stops at the `0` chunk and treats the remainder as the next request's start.
- **TE.CL** — front-end honors `Transfer-Encoding`, back-end honors `Content-Length`. Craft a chunk that ends inside the CL boundary; everything after CL's cutoff becomes the smuggled prefix.
- **TE.TE** — both support TE, but one can be tricked into *not seeing it* via obfuscation: `Transfer-Encoding: xchunked`, `Transfer-Encoding : chunked` (space before colon), `Transfer-Encoding:[tab]chunked`, wrapping/folding, duplicate TE headers with different values, `Transfer-Encoding: chunked` + `Transfer-Encoding: cow` (some servers take first, some last, some concatenate).

Why browsers don't hit this accidentally: browsers don't send chunked request bodies, and Burp auto-normalizes chunked encoding in Repeater — **turn off "Update Content-Length" and use HTTP/1.1** when probing.

## Detection signals

Probe with requests engineered so a vulnerable server produces an *observable* anomaly — never ambiguous requests that could poison a real user's request:

- **CL.TE timing probe**: send a small CL with a TE body that leaves a partial request hanging (e.g. body `0\r\n\r\n` followed by a partial line). Back-end waits for the rest → response delay/timeout vs baseline.
- **TE.CL timing probe**: send `Content-Length` *larger* than the de-chunked body; the back-end reads CL bytes and waits for bytes that never arrive → delay. (The other direction — CL *smaller* than the body — is the attack itself: the back-end stops early and the leftover becomes a smuggled prefix → differential response on the next request.)
- **Differential responses**: smuggle a complete second request to a known endpoint; the *next* normal request (yours, via Repeater) returns the smuggled endpoint's response or a 404/unrecognized-method error like `"GPOST"`.
- Use a non-existent method prefix (`GPOST`) so a confirmed desync produces an obvious error — and can't match a real endpoint, limiting blast radius.

Confirm before exploiting: repeat probes, check the front-end isn't just rejecting TE, and verify on a second connection that the *response* anomaly follows the smuggle, not random noise. Front-ends that drop/ normalize `Transfer-Encoding` or error on ambiguous requests are not exploitable — note and move on.

## Exploitation

Each lab maps to a primitive:

- **Bypass front-end security controls** — smuggle a request to a blocked path (`/admin`): front-end sees the allowed outer request, back-end processes the smuggled admin request.
- **Reveal front-end rewriting** — smuggle a POST to an endpoint that reflects a parameter; the response echoes headers the front-end injected (`X-Forwarded-For`, `X-Forwarded-Host`, session cookies). Replay the smuggle *with* those headers to reach restricted functionality.
- **Bypass client authentication** — where TLS client certs are terminated at the front-end and forwarded as a header, smuggle a request with that header to impersonate.
- **Capture other users' requests** — smuggle a partial request whose body lands in a stored location (comment field). Victim's next request completes yours; their headers (cookies) get stored → read them. Needs a comment-like feature + enough of their request captured.
- **Exploit reflected XSS via smuggling** — smuggle a request containing the XSS in a header the front-end validates (e.g. `User-Agent`); victim's request gets your reflected-XSS response with *no user interaction needed* — kills the "self-XSS / unexploitable reflected XSS" objection.
- **On-site redirect → open redirect** — smuggle a request to a path that 301s using the Host header; the next requester's request for a JS/CSS asset gets redirected to your host → import/poison their page load.
- **Web cache poisoning via desync** — smuggle a request that produces a poisonable response for a keyed URL other users fetch.
- **Web cache deception via desync** — smuggle so victim's sensitive response is stored under a cacheable key.

Prevention knowledge for reports: HTTP/2 end-to-end (no downgrade), disable back-end connection reuse, front-end must normalize ambiguous requests, reject messages with both CL and TE.

## Lab reference

`https://portswigger.net/web-security/all-labs#http-request-smuggling` — labs progress: CL.TE → TE.CL → TE.TE/obfuscation → confirm → each exploitation primitive above. Advanced variants (HTTP/2, CL.0, tunnelling) → `ch02-http2-desync.md`.
