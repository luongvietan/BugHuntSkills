# Ch8: Mobile, Auxiliary "Noise" Vulns & the Fast-Testing Checklist

Source: `10_Mobile` + `11_Auxiliary_Info` + `Fast Testing Checklist`. Wrap-up stage: mobile storage, the low-severity classes that still report, and Haddix's time-boxed assessment for maximum results in a fixed window.

## Mobile — data storage first

Mobile apps commonly fail to encrypt files that store PII. Where to look for unencrypted PII:

- Phone system logs (readable by other apps on some versions)
- WebKit cache (`cache.db`)
- plists, SQLite dbs, other app files
- Hardcoded inside the binary itself

Quick iOS spin-up: Daniel Meyer's **idb** (era tool — modern equivalent: MobSF/Objection/Frida; see `owasp-mas` skill for depth). Also check what the app sends: analytics/beacons leaking PII ties into the transport stage (ch07).

## Auxiliary — the vulns formerly known as "noise"

Lower-severity classes that are still worth reporting (and sometimes required for chains):

- Content spoofing / HTML injection
- Referer leakage (sensitive params leaking to third parties)
- Missing security headers
- Path disclosure
- Clickjacking
- ++ (anything the program's policy counts — informational findings vary by program)

## The "n-minute" data-driven assessment — diminishing returns, weaponized

Haddix's ordered battery for extracting maximum results from a fixed time window. Each step is cheap and hits the historically highest-yield spots:

1. Hit **search, registration, contact, password-reset, and comment forms** with polyglot strings (ch05).
2. Point Burp's scanner (or equivalent) at those same specific functions.
3. **Cookie lifecycle**: check cookie → log out → check cookie → log in → check cookie → replay the *old* cookie and see if it still grants access.
4. **User enumeration** on login, registration, and password reset.
5. **Do a password reset** and observe: plaintext password delivered? URL-based token? Predictable token? Token reusable? Auto-login after reset?
6. **Rotate numeric account identifiers** anywhere they appear in URLs → context change / IDOR.
7. Take the **security-sensitive functions/files** and test: unauthenticated browsing, lower-priv browsing, CSRF + CSRF-protection bypass, and whether they work over HTTP.
8. Directory brute-force with a **top shortlist** (SecLists) — not the full wordlist.
9. Check **upload functions** for alternate executable file types (XSS or server-side code).

## Fast Testing Checklist — the full task grid

TBHM ships a merged checklist (Haddix's methodology × Web Application Hacker's Handbook task list). Condensed map — use it as the completeness audit after tactical testing:

| Section | Covers |
|---|---|
| App recon & analysis | manual content map, hidden/default content via brute-force, debug params, data entry points, tech fingerprint + CVE research, stack-matched wordlists, spidered surface, JS file inventory |
| Access handling | authn: password rules, username enum, guessing resilience, recovery, remember-me, impersonation, uniqueness, cred distribution, fail-open, multi-stage; session: token meaning/predictability/transmission/disclosure, session mapping, termination, fixation, CSRF, cookie scope; authz: control requirements, multi-account tests, insecure methods (params, Referer) |
| Input handling | fuzz all params; SQLi; reflected data/XSS; header injection; arbitrary redirect; stored attacks; OS command injection; path traversal; script injection; file inclusion; SMTP, SOAP, LDAP, XPath injection; native flaws; SSRF in all redirecting params |
| Application logic | logic attack surface, client-side trust, thick-client components, multi-stage process flaws, incomplete input, trust boundaries, transaction logic |
| Hosting | shared-infrastructure segregation, web-server vulns: default creds/content, dangerous HTTP methods, proxy functionality, vhost misconfig, server software bugs |
| Misc | DOM attacks, frame injection, local privacy (persistent cookies, caching, sensitive data in URLs, autocomplete forms), info-leak follow-up, weak SSL ciphers |

The full checkbox version lives in the source repo (`Fast Testing Checklist.md`); `owasp-wstg` skill is the modern checklist authority if you need per-test procedures.
