# API8:2019 — Injection

> **2023 status:** dropped from the 2023 top 10 — still a real test class.
> Payload depth: `payloads-all-the-things`; mechanism depth: `web-security-academy`.

## Core Idea
Untrusted data reaches an interpreter (SQL, NoSQL, LDAP, OS command, XML parser, ORM) as part of a command/query. APIs take injection input anywhere: params, body, headers, integrated upstream services. Scanners and fuzzers find these easily; impact runs to full host takeover.

**Scores**: Exploitability 3 · Prevalence 2 · Detectability 3 · Technical Impact 3

## Is the API Vulnerable? (Test Conditions)

- Client data **not validated/filtered/sanitized** before reaching an interpreter.
- Client data **concatenated** into SQL/NoSQL/LDAP/OS-command/XML/ORM calls.
- **External-system data** (integrated services) treated as trusted — also injectable.

Injection points: query params, path segments, JSON body fields, headers, multipart fields — any input reaching an interpreter.

## Example Attack Scenarios

**Scenario 1 — command injection via multipart field**: firmware endpoint `/api/CONFIG/restore` passes `appId` straight into `snprintf(cmd,...); system(cmd)`. Exploit: `curl -F 'appid=$(/etc/pod/power_down.sh)'` → executes shutdown on every device with that firmware. *Firmware/hardware APIs concatenate into shell — always test for `$(...)`, backticks, `;|&`.*

**Scenario 2 — NoSQL operator injection**: `DELETE /api/bookings?bookingId=678` maps to `Bookings.findOneAndRemove({_id: req.query.bookingId})`. Send `bookingId[$ne]=678` → `$ne` operator matches *all other* records → delete another user's booking. **`[$ne]`, `[$gt]`, `[$regex]` param-array operators are the canonical NoSQL-injection trick.**

## How To Prevent

- Keep data separate from commands/queries; validate via a single trusted maintained library.
- Validate/filter/sanitize all client + integrated-system data.
- Escape special chars with the target interpreter's syntax; prefer parameterized APIs.
- Limit returned record counts (caps injection-based mass disclosure).
- Strict data types and patterns for every string param.

## Anti-patterns

- **Only testing `'` SQLi**: APIs are JSON — try object/array operators (`[$ne]`), type juggling, nested params.
- **Trusting internal integrations**: upstream-service data is still untrusted input.
- **Missing the second-order sink**: injected value stored then used later (e.g., mass-assigned param reaching `system()` — see API6 scenario 2).

## Key Takeaways

1. Fuzz every input class: path, query, JSON fields (string + array + object forms), headers, multipart.
2. NoSQL: `param[$ne]=x`, `param[$gt]=`, `{"$regex":".*"}` in JSON bodies.
3. Command injection probes: `$(cmd)`, `` `cmd` ``, `;cmd`, `|cmd`, `&&cmd` — look for OS-backed features (convert, restore, import, ping).
4. ORM doesn't immunize — raw-query fallbacks and unsafe filters still inject.
5. Chain thinking: injection + no record limit = mass disclosure; injection via mass assignment = stored RCE.

## Connects To

- **API6 (ch06)**: mass assignment delivers params into server-side interpreters
- **API4 (ch04)**: unlimited record counts amplify injection disclosure
- OWASP Injection/NoSQL/Command Injection cheat sheets; CWE-77, CWE-89
