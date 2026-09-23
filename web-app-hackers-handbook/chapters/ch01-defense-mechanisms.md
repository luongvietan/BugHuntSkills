# Ch1–2: Web Application (In)security & Core Defense Mechanisms

Source: Chapters 1–2. The core security problem: a user can submit **arbitrary input** from outside the application's control — crafted request parameters, altered request sequences, non-browser tooling. Every defense is a server-side mechanism compensating for that fact. The big three interlock — **authentication** (who are you), **session management** (carrying identity across stateless HTTP), **access control** (what may you do) — and overall security equals the weakest link. Supporting mechanisms: **input handling**, **function handling** (malicious input reached privileged functions), **auditing/alerting**, and **management** interfaces (themselves attack surface).

## Mechanism → where it breaks

- **Input validation styles**: reject-known-bad (blacklists die on encodings/case/variants), accept-known-good (whitelist — correct but hard), sanitize (transforms input — vulnerable to escaping the escape character, truncation-after-escape), safe data handling (parameterized queries — best), semantic checks (verify meaning, not syntax).
- **Boundary validation**: validation must happen at **each trust boundary** the data crosses — web tier → app tier → DB → OS → browser — because "safe" is relative to the downstream interpreter. This is the book's canonical answer to "filter once at the perimeter."
- **Canonicalization problems**: the same logical input has many encodings (URL `%`, double-URL, HTML entities, Unicode variants, null bytes). Filters checking one representation miss another. Multi-stage decode-then-check vs check-then-decode ordering is a systematic bypass class.
- **Multistage validation**: validation step N modifies data, invalidating step N−1 (e.g., strip `<script>` → later decode → `<script>` reappears). Defenses must be applied in a known order to a fully decoded string.

## Attack approach

- For every input, ask: **which boundary does this cross next?** A string safe for the app tier may be a metacharacter at the shell, SQL, XML, SMTP, or DOM boundary.
- Probe defenses by **isolating the filter's trigger**: submit payload fragments one at a time until you know exactly which character/expression is blocked — then attack that specific check (see ch11 XSS filter catalog).
- Map the defense's **scope**: a global WAF rule often covers only query strings or only GET — POST body, cookies, headers, path segments, and second-order paths may bypass it entirely.

## Defense/evasion table

| Defense | Canonical failure |
|---|---|
| Blacklist `script`, `select` | case mix, `<scr<script>ipt>`, alternate tags/functions, encodings |
| Length truncation | truncation *after* escaping re-opens injection (odd-quote trick, ch11) |
| Character escaping (quotes → `''`, `\`) | escape char itself unescaped → `foo\';` breaks out |
| URL-decode then filter | double-encode: `%253c` survives first decode, becomes `<` at second |
| HTML-encode output | reflection inside event handler/JS string — HTML-encoding is attacker-usable (`&apos;`) |

## Checklist

- [ ] Enumerate every defense: input filters, session handling, access control points, WAF, rate limits.
- [ ] For each: name the check → identify its representation/ordering → pick the canonical bypass.
- [ ] Verify the big three interlock: does auth enforcement actually precede the action? Are tokens bound to sessions? Is every function's access check server-side?
