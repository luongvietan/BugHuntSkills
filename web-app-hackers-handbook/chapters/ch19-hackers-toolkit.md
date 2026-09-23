# Ch20: A Web Application Hacker's Toolkit

> **Edition note (2011):** tool *roles* are durable (proxy sees all, scanner
> improvises nothing); specific versions are historical — current stack:
> Burp Suite Pro + extensions (Param Miner, Turbo Intruder), Caido/ZAP,
> ffuf, httpx, Nuclei. Recon tooling currency: `recon-pipeline`.

Source: Chapter 20. The toolset is small: browser + intercepting proxy + (optionally) scanner + custom scripts. What matters is *what each tool is for* and *where each one fails* — a scanner that improvises nothing and a proxy that sees everything complement each other.

## Browsers

- IE (legacy app compatibility + zone model matters — Trusted Sites changes XSS impact), Firefox (extensions: WebDeveloper — View Generated Source shows what the parser actually built from broken HTML; Live HTTP Headers-style replay), Chrome (devtools, modern default). Pick per target tech.

## Integrated testing suite (Burp model)

- **Intercepting proxy**: the core — full request/response control, SSL interception (cert warnings are the price), invisible passthrough for non-proxy-aware clients, upstream proxy chaining. Rules of engagement: *all* app traffic flows through it → it becomes your complete map and replay bench.
- **Manual request tools**: Repeater (edit-resend loop for parameter surgery — the primary hunting tool), Decoder (encode/decode chains), Comparer (response diffs).
- **Spider/mapper**: builds the content tree; never trust it alone for JS/multistage flows.
- **Sequencer**: statistical randomness of tokens (ch06).
- **Intruder**: positioned payload automation (ch13).
- **Session-handling rules/macros**: automate re-login, nonce/CSRF refresh so automated tools keep working (ch13).
- **Extensions**: Scanner, custom Burp Extender plugins, third-party modules.

## Standalone vulnerability scanners

- **What they catch**: broad, known-signature classes — reflected XSS, basic SQLi, traversal, command injection, some config issues; good at *coverage* of huge param space.
- **Inherent limits** (know these — they define your manual work): no improvisation (business logic invisible), no intuition (can't decide *which* anomaly matters), struggle with: multistage flows, state/context, custom auth/session schemes, nonstandard input channels (cookies, headers, serialized data), second-order paths, novel/filtered variants, session-terminating side effects (scanner logs itself out, DoSes the app).
- **Dangerous effects**: scanners change state — mass junk data, lockouts, destructive requests; run on test instances or with scope limits, never blind against prod.
- **Usage discipline**: scan *after* manual mapping (so you know what it missed); use per-function scans over full-site sweeps; verify every hit manually (false-positive rate is real); compare against your manual coverage list — unscanned surface is your list.

## Alternatives & supplements

- **Non-proxy-aware clients**: browser plugins that tunnel through the proxy, transparent proxy modes.
- **Firebug/devtools**: client-side debugging — DOM-XSS, JS inspection, breakpoint the validation.
- **Custom scripts**: when tools don't fit the workflow — bespoke request loops, nonces, parsing (ch13); small and resumable beats elegant.
- **Stunnel/SSL helpers** for non-HTTPS-aware tools.

## Method

Proxy captures everything → manual walkthrough maps app → targeted Repeater work per hypothesis → Intruder/Sequencer for scale → scanner for breadth-coverage audit on already-mapped surface → custom script where workflow defeats tools → verify every automated hit by hand.

## Checklist

- [ ] All traffic through proxy; upstream/SSL configured.
- [ ] Session-handling rules cover auth/nonce needs before automation.
- [ ] Scanner used for coverage on mapped surface only; every hit hand-verified.
- [ ] Client-side issues debugged in devtools, not guessed.
- [ ] Custom scripts logged, resumable, throttled.
