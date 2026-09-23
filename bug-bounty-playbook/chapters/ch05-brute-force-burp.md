# Ch 5 — Brute Forcing + Burp Suite

## Brute forcing

**Rule of thumb: "if there is a login screen it should be brute forced."** Passwords are the weak point — guessable, reused, or left as defaults.

Requirements: (1) login endpoint, (2) username(s), (3) password list.

**Endpoints to hit**: web login pages (Outlook, VPN, router/firewall admin, wp-admin), SSH :22, RDP :3389, VNC :5900, FTP :21, Telnet :23.

**Default credentials** — the author's most underrated win: SecLists `Passwords/Default-Credentials` holds vendor defaults for hundreds of devices. Look up the vendor → try all its defaults. People forget to change them constantly. For other services, Google `<service> default credentials`.

**Tooling**: `hydra` (thc-hydra) — broad protocol support. For ≤5 guesses do it by hand; otherwise automate.

## Burp Suite — the one mandatory tool

*"Almost every exploit you pull off will be in Burp."* Buy Pro; Community works but throttled.

### Proxy / HTTP history (80% of your time)

- Set proxy listener → point browser at it → import Burp CA cert. Keep `intercept` **off**; work from HTTP history.
- **Read traffic by instinct**: POST → stored XSS/CSRF; URL containing email/username/id → IDOR; JSON MIME → backend API; `url=`/`redirect=`/`callback=` params → SSRF/open-redirect/SOP bypass.
- **Search across all history** — the author's vulnerability finder: search `url=` → SSRF/redirect candidates; `Access-Control-Allow-Origin` → SOP/CORS checks; `callback=` → JSONP/SOP bypass.

### Target (site map + scope)

- Site map auto-builds a per-host tree — essential for mapping **undocumented API endpoints**.
- Scope limits scanning to in-scope domains.

### Intruder (fuzzing / brute force)

- Right-click request → Send to Intruder → Clear positions → highlight value → Add.
- **Attack types**: Sniper (one list, one position at a time) · Battering ram (one list, all positions simultaneously) · Pitchfork (per-position lists, zipped) · Cluster bomb (all combinations).
- Payload lists: Burp defaults or SecLists `Fuzzing/` → Start attack → triage by response length/status.
- Pros actually use **Turbo Intruder** plugin — faster and more powerful.

### Repeater (manual replay)

- Right-click → Send to Repeater → modify → Send → inspect response.
- **Rename tabs to findings** (e.g. `SSRF`) — instant organized record of interesting requests.
