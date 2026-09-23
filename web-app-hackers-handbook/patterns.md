# Patterns — Reusable Heuristics from the WAHH

Cross-cutting heuristics that apply across vuln classes. Per-class mechanics live in `chapters/`.

## The signature loop (per vulnerability class)

```
Mechanism → Attack → Defense/evasion → Hack-steps checklist
```

Understand *what the server does with the input*, derive the *assumption it encodes*, violate the assumption precisely, then exhaustively enumerate (parameters × encodings × stages × accounts).

## Universal heuristics

- **The client is hostile territory**: every request element is attacker-controlled — params (present/absent/duplicated/reordered), headers, method, sequence, timing, content-type. Server trust in any of these is the bug.
- **Name the check, pick its bypass**: defenses fail canonically — substring blacklists (case/nesting/encoding), validation-when-present (drop the param), presence-vs-correctness (supply a different valid shape), wrong-scope storage (static for per-user), wrong-order operations (decode↔validate↔truncate). Identify which failure *mode* the defense has.
- **Decode order is everything**: for any filtered payload ask "what does the filter see vs what does the interpreter get?" Double-encoding, multibyte lead bytes, HTML entities inside attributes, null bytes — the interpreter's view wins.
- **Escape the escape**: any sanitizer that escapes metachars must escape the escape character first. `foo\;` turns their defense into your separator. Same for quote-doubling: an odd number of quotes after truncation un-escapes the string.
- **Absent ≠ empty**: removing a parameter entirely hits different code than `param=`. Presence-based role checks (password-change ex) fall to deletion, not blanking.
- **Two accounts, always**: horizontal and vertical diffs of the same request are the cheapest bug-finder in the book. Third account = unauthenticated baseline.
- **State has scope; abuse the scope**: per-user data in static/shared storage → cross-user leaks and races; session data written by feature A steering feature B → privilege/identity pollution. Ask where each piece of state lives and who else can write it.
- **Workflow order is an assumption**: multistage processes assume sequencing. Skip, repeat, reverse stages; submit stage-N params at stage-M; complete nothing and see what the final step accepts.
- **Counts and deltas are oracles**: match counts, response lengths, timing, status codes, new cookies — any deterministic difference extracts a bit per request. Inference defeats "they can't see the data."
- **Every decode/serialize boundary is a trust boundary**: remoting blobs, ViewState-style opaque values, XML/SOAP envelopes, JSON, AMF, custom serialization — decode everything opaque; the boundary is where validation is missing.
- **The interpreter defines dangerous**: a character is only dangerous at a specific boundary — `'` for SQL, `<` for HTML, `;` for shells, `|` for LDAP, `../` for filesystems, CRLF for headers. Build a per-boundary metachar map, then fuzz per boundary.
- **Errors are documentation**: provoked errors reveal code paths, query shapes, file paths, versions. Deliberately break things at every layer — each subsystem's error is a fingerprint.
- **Coverage beats cleverness**: the HACK STEPS discipline — every param, every stage, both methods, all encodings, admin surfaces too. The untested parameter holds the bug.

## Defense-side smells (what to hunt *for*)

- Blacklists/regexes instead of parameterized handling.
- Validation that runs on some inputs but not equivalent ones (query but not body, GET but not POST, ASCII but not multibyte).
- Sanitizers applied in a chain where step order isn't defended.
- Crypto where any oracle exists (reveal or encrypt).
- Audit/debug output keyed to static storage or reachable URLs.
- Numeric business limits without sign/canonicalization checks.
- Discounts/prices/limits computed before the order is final.
- Search/index functions spanning data above the requester's privilege.
- Shared components processing *all* submitted params, not just expected ones.

## Attack-chain accounting

- Token theft (XSS) → session hijack → everything the user can do.
- Open redirect → OAuth/SAML token capture, SSRF pivot, phishing realism.
- Enumeration → targeted brute force → account → deeper surface.
- Info leak (stack trace, debug store) → internal structure, creds, tokens → escalation.
- Second-order/stored injection → admin compromise — stored payloads deliberately aimed at admin-facing surfaces.
- Cross-tenant discriminator swap → whole-customer data in shared apps.
- File write (traversal/WebDAV/LFI+upload) → code exec on host.

## Anti-patterns the book warns about

- Filtering once at the perimeter instead of validating at each boundary.
- Trusting UI-hidden fields/disabled controls/absent parameters as "unsubmittable."
- Believing a WAF makes the app safe — it's a signature layer, bypassable by construction.
- Treating scanners as coverage — they can't do logic, multistage, state, or novelty.
- Reporting a payload instead of the mechanism+impact ("XSS" vs "stored XSS in admin log viewer → privileged action").
- Stopping at "no output in response" — timing and OOB channels still carry data out.
