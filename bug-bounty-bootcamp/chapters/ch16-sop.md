# Ch19: Same-Origin Policy Attacks

Source: Chapter 19. SOP = browsers isolate `scheme://host:port` origins — page A can't read responses of origin B. Apps *relax* SOP for legit reasons; each relaxation mechanism is an attack surface.

## CORS (Cross-Origin Resource Sharing)

Server says `Access-Control-Allow-Origin: X` → browser lets origin X read responses. Misconfig patterns to probe with an `Origin:` header:

| Response | Meaning | Exploit |
|---|---|---|
| `ACAO: <your-origin>` + `ACAC: true` | reflects any origin + credentials | full authenticated read from attacker page |
| `ACAO: null` | `null` origin trusted | `<iframe sandbox>` makes origin `null` → read |
| `ACAO: *.example.com` suffix-match | `evilexample.com` or `example.com.evil.com` passes | subdomain/prefix confusion |
| `ACAO: evil.example.com` needed | subdomain allowed | find XSS/subdomain-takeover on any subdomain → read API |
| `ACAO: https://` only | scheme checked, host not | MITM/plain-HTTP downgrade paths |

Test methodically: `Origin: https://evil.com`, `Origin: null`, `Origin: https://target.com.evil.com`, `Origin: https://eviltarget.com`, `Origin: http://sub.target.com` — then check `Access-Control-Allow-Credentials`.

## postMessage

Cross-origin iframe/window messaging. Bugs: receiver doesn't check `event.origin` (→ send crafted message → XSS/actions), or sender posts secrets to `'*'` (→ open an attacker iframe/popup, listen, steal data). Grep JS for `addEventListener("message"` and `postMessage(` — audit both sides.

## JSONP

Endpoint wraps data in a callback: `GET /api/me?callback=cb` → `cb({…secret…})`. Any site can `<script src="…?callback=steal">` and read the data cross-origin — effectively a sanctioned CORS bypass. Hunt JSONP endpoints (`callback=`, `jsonp=`, `cb=`), check they require auth cookies (they usually do — that's the bug).

## document.domain & others

- `document.domain = "example.com"` relaxation: any subdomain can claim parent domain → subdomain XSS = parent-app compromise.
- Cookies scoped `.example.com` (leading dot) shared to all subdomains → subdomain takeover/XSS → session theft.
- **XSS itself** runs in the page's origin → bypasses SOP entirely within that origin.

## First-bug checklist

List endpoints returning sensitive data → probe `Origin:` reflections incl. `null` → check `ACAC` → grep JS for postMessage/JSONP → check cookie domain scope → PoC = attacker page reading victim's authenticated response.
