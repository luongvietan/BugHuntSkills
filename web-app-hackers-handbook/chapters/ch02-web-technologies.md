# Ch3: Web Application Technologies

Source: Chapter 3. The substrate every attack is expressed in. Know HTTP requests/responses cold, then the layers where interpretation diverges — divergence between parser and filter is where vulnerabilities live.

## HTTP essentials for attackers

- **Methods**: GET/POST/HEAD/PUT/DELETE/OPTIONS/TRACE — applications often enforce checks per-method; same action via different method is a canonical bypass (see ch07, ch12). TRACE echoes the request (historically useful for XSS via headers).
- **Key headers**: `Referer` (leaks URLs/tokens; attacker-controllable), `User-Agent` (XSS sink in admin log viewers), `Host` (vhost selection; reset-link poisoning), `Cookie`/`Set-Cookie` (domain/path/secure/HttpOnly scope), `Content-Type` (parser selection — `application/x-www-form-urlencoded` vs `multipart/form-data` vs `text/xml` reach different code), `X-Forwarded-For` (often trusted for IP-based controls — spoofable).
- **Status codes**: 301/302 redirects (Location header targets), 401/403 (auth boundary map), 500 (error/injection signal). Deltas across these are your oracle.

## Encoding — the bypass alphabet

- **URL encoding**: `%3c` — first decode is server's; watch for a *second* decode inside the app.
- **HTML entities**: `&lt;`, `&#x3c;`, `&#60;` (decimal, hex, leading zeros, optional `;`) — decoded by the *browser* inside attribute values before JS runs.
- **Base64/hex** — remoting payloads and opaque params; decode every blob you see.
- **Unicode/UTF variants**: UTF-7 (`+ADw-`), UTF-16, US-ASCII high-bit tricks, overlong encodings, multibyte sets (Shift-JIS `0xf0` lead byte eats the following quote — see ch11).
- **Null byte `%00`**: terminates strings in native code — WAFs/extensions checks see only the prefix.

## Client-side stack

- **Same-origin policy**: origin = scheme+host+port; cookies/scripts/DOM are origin-scoped. The entire XSS/CSRF threat model derives from what the SOP does and doesn't cover (SOP doesn't stop *sending* cross-origin requests — only reading responses; hence CSRF).
- **Ajax/JSON**: asynchronous partial-page updates multiply entry points; JSON responses still need encoding for their insertion context.
- **Cookies**: scope via `domain`/`path`; `secure`, `HttpOnly` flags are defensive claims to verify, not assume. Cookie-scope mistakes = cross-subdomain session theft.
- **Remoting & serialization**: Java/Flash/Silverlight remoting (AMF), custom serialized blobs, web services/SOAP. Anything opaque in params = decode it; serialization boundaries are trust boundaries where validation is usually missing.

## Checklist

- [ ] For every request: which headers does the app actually read (Referer, XFF, Host, Content-Type)?
- [ ] Inventory every encoding in use; for each decode step, note what filter ran before vs after.
- [ ] Identify all non-HTML content types returned/served and every remoting/serialization channel.
- [ ] Note cookie flags and scope for every token-bearing cookie.
