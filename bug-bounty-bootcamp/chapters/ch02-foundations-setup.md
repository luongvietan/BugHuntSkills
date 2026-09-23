# Ch3-4: How the Internet Works & Environment Setup

Source: Chapters 3-4. Foundation material: HTTP mechanics and the Burp-based testing environment.

## HTTP essentials (what matters for hacking)

- **URL anatomy**: `scheme://host:port/path?query#fragment`. The query is attacker-controlled input; the host determines SOP origin (scheme+host+port); path manipulation drives traversal and routing bugs.
- **Requests**: method + path + headers + body. Methods map to intent — GET reads, POST creates/changes, PUT/PATCH updates, DELETE deletes. Always test whether the *server* enforces the intended method semantics.
- **Responses**: status code + headers + body. Status-code classes double as a fuzzing signal (2xx found, 3xx redirect chain, 401/403 access control, 5xx possible injection). Security-relevant headers to always check: `Set-Cookie` flags, `X-Frame-Options`/`frame-ancestors`, `Access-Control-Allow-Origin`, `Content-Security-Policy`, `Location`.
- **Cookies & sessions**: `HttpOnly` (JS can't read → limits XSS impact), `Secure` (HTTPS only), `SameSite` (Strict/Lax/None → drives CSRF feasibility). Session state lives server-side keyed by the cookie — stealing/fixating the cookie = stealing the session.
- **Keep-alive & encoding**: requests are text — any field can be edited via proxy regardless of what the browser UI allows.

## The testing environment

- **Burp Suite Community**: proxy (intercept/edit/replay), repeater (manual re-issue), intruder (fuzzing — throttled in free tier), decoder, comparer, sequencer (token entropy). Extensions worth installing: Autorize, AuthMatrix, Auto Repeater (access-control testing).
- **Setup**: browser → proxy 127.0.0.1:8080 (FoxyProxy or per-profile), install Burp CA cert to intercept HTTPS. Never proxy through public networks.
- **Browser devtools**: Inspector for DOM/context, Console for JS errors, Network tab for finding API calls the UI hides, Debugger for stepping through client-side validation and DOM XSS sources/sinks.
- **Emulators/practice targets**: DVWA, OWASP Juice Shop, PortSwigger Web Security Academy, HackTheBox — legal places to learn each bug class before hunting live.
- **Useful extras**: CyberChef (encode/decode chains), curl for scripting, a dedicated test browser profile, and throwaway accounts for every program.
