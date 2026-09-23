# Patterns — Reusable Heuristics from OWASP MASTG

Cross-cutting heuristics across the MASVS categories. Per-category detail lives in `chapters/`.

## The master test shape (every MASTG-TEST)

```
Weakness (MASWE) → Overview → Steps → Observation → Evaluation
                static: "references to X APIs"   |   dynamic: "runtime use of X"
```

Apply it to any mobile check: name the weakness, grep the config/code for the mechanism, then exercise the app until the code path runs, observe artifacts (files, traffic, logs, calls), and decide pass/fail against the stated condition — not against "the scanner flagged it".

## Universal heuristics

- **Manifest/plist is the map**: `AndroidManifest.xml`, `Info.plist`, entitlements, `networkSecurityConfig`/ATS reveal every entry point before a single dynamic test — exported components, schemes, deep links, cleartext exceptions, debug flags.
- **Exported ⇒ untrusted**: every exported component, intent extra, deep-link param, provider URI arg, pasteboard value, and WebView input is attacker-controlled data. Trace each to a sink (query, path, bridge call, navigation, parser).
- **"References + runtime" doubles confidence**: a static API hit without runtime reachability is a candidate; runtime behavior without code context is unexplainable. Confirm both.
- **Sensitive data is contractual**: findings hinge on the agreed data definition — classify before testing; a "leak" of non-listed data is informational.
- **Crypto findings are purpose-bound**: same algorithm, different verdicts — MD5 for file-integrity vs MD5 for password storage; weak RNG for a shuffle vs for tokens. Always name the security-relevant context.
- **Boolean security checks are bypassable**: any client-side decision (root detected? biometric ok? cert pinned?) implemented as a return value can be flipped by hooking. Strong designs bind the check to cryptography (keystore unlock, attestation verified server-side).
- **The client lies to the server by nature**: anything enforced only in the app (validation, authZ hints, price, feature flags) is server-testable — mobile clients are more attacker-modifiable than web ones.
- **Defense-in-depth is measured in effort**: rate resilience controls by time-to-bypass (stock check = minutes, custom layers = days); the reverse engineer always wins eventually.
- **Every storage sink has three twins**: files, logs, and backups — a secret cleaned from one usually survives in the others; check all three plus keyboard cache and snapshots.
- **Transport findings come in pairs**: cleartext path + validation path — cleartext spotted on the wire means nothing until you also know whether the app would have accepted a rogue cert.

## Workflow patterns

- **Two builds**: release (verify controls) + debug (cover the rest) — ask for both; coverage halves without the debug build.
- **Snapshot-diff**: baseline the data dirs, run the feature, diff — new files are where storage findings live.
- **Proxy first, hook second**: get plaintext visibility via proxy+CA; escalate up the MITM ladder (pinning bypass → TLS hooks → repackage) only as far as needed.
- **MobSF for breadth, hands for depth**: automated scan seeds a checklist; manual confirmation gives each finding exploit context and kills false positives.
- **Grep vocabulary per category**: storage APIs, crypto `getInstance`/CommonCrypto, `BiometricPrompt`/`LAContext`, `TrustManager`/`URLSessionDelegate`, `addJavascriptInterface`, `getInstance("DexClassLoader")` — build the keyword list once, reuse per app.
- **Two accounts still apply**: backend API authZ bugs (IDOR/BOLA) are found exactly as in web — mobile is often the *less* tested client for the same endpoints.
- **Old versions carry old bugs**: test prior app versions and legacy API paths the app still calls.

## Anti-patterns the guide warns about

- Reporting scanner output without an exploit scenario (CSRF/reflected-XSS "mobile findings").
- Reporting absent pinning as a vulnerability at L1 — it's an L2 control.
- Counting a vulnerable dependency without proving the vulnerable path is reachable.
- Treating obfuscation/root detection as security controls rather than speed bumps.
- Defining "sensitive" after finding the leak — retrofitted severity.
- Dumping the keystore/user data beyond the minimal demonstration needed to prove the weakness.
