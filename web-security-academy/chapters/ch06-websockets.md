# Ch06: WebSocket Vulnerabilities

Source: `/web-security/websockets`. **GAP CLASS.** WebSockets are long-lived, bidirectional channels initiated by an HTTP `Upgrade` handshake — after the handshake there are no CSRF tokens, no Same-Origin Policy on messages, and input-validation assumptions from HTTP often don't transfer.

> **Lab vs live (bounty-safe):** message-level probes on your own socket are
> safe. CSWSH (cross-site WebSocket hijacking) is a victim-delivery attack —
> prove it against your own session from your own page; never deploy a
> working exploit page against real users.

## Mechanism

Handshake: client sends `GET /chat HTTP/1.1` with `Upgrade: websocket`, `Connection: Upgrade`, `Sec-WebSocket-Key`, `Sec-WebSocket-Version`, optional `Sec-WebSocket-Protocol`; server answers `101 Switching Protocols`. Everything after is framed messages in both directions — usually JSON, frequently carrying the same actions/data as the HTTP API.

**Why it's a distinct class**: virtually any HTTP vuln can live inside WS messages (XSS in chat, SQLi in query params, XXE in XML payloads, blind bugs via OAST), plus WS-specific flaws in the handshake itself.

## Manipulating WS traffic (Burp)

- **Proxy → WebSockets history tab**: find that WS is in use, watch message flow.
- **Intercept**: toggle client→server / server→client interception in Proxy settings; edit messages in flight.
- **Repeater for WS**: send a message to Repeater → replay, edit, generate new messages in either direction; history panel shows the full socket transcript.
- **Handshake manipulation**: pencil icon next to the WS URL in Repeater → attach to an existing socket, **clone** a connected one, or reconnect a dropped one — edit the handshake request (cookies, tokens, headers) before connecting. Use when: reaching more attack surface, your socket was dropped mid-test, tokens went stale.

## Vuln patterns

- **Message-content bugs**: send the classic probes as WS message values — `{"message":"<img src=1 onerror=alert(1)>"}` for chat XSS (the receiving user's browser renders it — stored-XSS-equivalent over WS), `'`-probes for SQLi, XML with entities for XXE, blind payloads + OAST listener when nothing reflects.
- **Handshake trust bugs**: server makes security decisions on handshake headers — e.g. trusts `X-Forwarded-For` for IP-based access control/blocking; reconnect with a spoofed `XFF` to bypass blocks or reach restricted functions.
- **Cross-site WebSocket hijacking (CSWSH)** — the signature WS bug: the handshake relies *only* on session cookies for auth. WebSocket handshakes are not protected by SOP, so an attacker page on any origin can open a WS to the target *with the victim's cookies* (SameSite=Lax doesn't block WS in older contexts; no `Sec-WebSocket-*` CORS-equivalent exists unless the server validates `Origin`). If the server doesn't check `Origin`, the attacker's JS gets a fully authenticated socket → read the victim's private messages, send actions as them.
  - **Test**: from a different origin (your own page or modified `Origin` header), replay the handshake with the victim-like cookie absent/foreign — if the socket still opens and the server never validates `Origin`, it's CSWSH. Check whether the socket works with *no* cookies too (unauthenticated WS exposing data = separate finding).
- **Securing (for remediation)**: `wss://` only, validate `Origin` server-side, treat all message content as untrusted input, CSRF-token-equivalent in the handshake for state-changing channels, session-bound tokens in handshake rather than cookies alone.

## Lab reference

`https://portswigger.net/web-security/all-labs#websockets` — three labs: message XSS, handshake header manipulation, CSWSH.
