# Ch14: Automating Customized Attacks

> **Edition note (2011):** the "detect the hit, respect the session machine"
> doctrine is durable; tooling has moved — Turbo Intruder (incl. single-
> packet race), ffuf/feroxbuster, Nuclei, and scripted Burp extensions are
> the current automation stack. Any high-volume fuzzing needs explicit
> policy permission + rate caps.

Source: Chapter 14. Manual probing finds the *mechanism*; automation multiplies it — enumerate identifiers, harvest data, fuzz parameters. The hard part isn't sending requests, it's **detecting hits** and **staying inside the app's session/state machine**.

## Uses

- **Enumerating valid identifiers**: usernames, doc IDs, file paths, session tokens, function names. Build candidate lists from observed formats (not generic wordlists — mirror the app's own naming scheme).
- **Harvesting data**: iterate a function that displays user/object data (profile by ID, search, directory) to aggregate datasets; match-count oracles (ch10 search ex) extract data the UI never displays.
- **Fuzzing**: systematic probes per parameter per vuln class — errors, injection, traversal, XSS strings — driving off *anomalies* not just hits.

## Hit detection — the engineering core

A script is only as good as its oracle. Monitor every channel:

- **HTTP status** (200/302/403/500 deltas), **response body** (error strings, content length, presence of the probe string, extracted markers), **`Location`** header (redirect targets differ by validity), **`Set-Cookie`** (new cookie = event occurred), **time delays** (deliberate `SLEEP`/slow-path confirmation; measure repeatedly — jitter lies), **server-side state changes** visible elsewhere (email received, log entry, second request's content).
- Calibrate a **baseline** first (known-invalid request's full response); a "hit" is a deviation, so log *all* responses and sort by each signal — don't hard-code one.
- **Fingerprinting strings**: when content differs subtly, grep for a signature token, count matches, or strip dynamic noise (tokens, timestamps) before diffing.

## Tooling

- **Burp Intruder**: position payloads precisely (multiple positions × attack type — sniper for one position, cluster-bomb for permutations); payload sets incl. recursive grep of responses, case-mutation, encoding; match/grep extract for hit signals; throttle/threads configured to target's tolerance.
- **JAttack-style custom scripting** (or Python + requests): needed when logic is nonlinear — token must be refreshed per request, response N feeds request N+1, signatures must be recomputed. The book's rule: *script the request loop, not the exploit* — keep the oracle readable and resumable (log progress, allow restart mid-range).

## Barriers & workarounds

- **Session handling**: tokens expire, per-request nonces, page-flow tokens (CSRF tokens fetched then submitted). Automation must: re-login on expiry detection, fetch-and-inject nonces each cycle, follow required request sequences (Burp session-handling rules/macros automate this).
- **CAPTCHA & human barriers**: test whether it's *enforced* (resubmit without solving, replay a solved pair, race the same answer, request via alternate channel/API); some implementations accept a fresh request session, expose the answer in the page/source, or have a static mapping. If genuinely strong — document it, don't burn the target.
- **Rate limits/lockouts**: slow to stay under thresholds (throttling is also courtesy — you are load on a live system); distribute across accounts/IPs only with authorization; design runs to pause on anomaly (lockout triggers are themselves findings).

## Method

1. Pick one function + one hypothesis ("IDs 1–5000 enumerate users").
2. Build candidate set from the app's own conventions.
3. Calibrate baseline; define 2–3 hit signals.
4. Instrument the request loop: auth, nonce refresh, throttle, logging, resume.
5. Dry-run 20 requests; verify the oracle separates hits from misses; then scale.

## Checklist

- [ ] Baseline captured; hit signal verified against known-good/known-bad cases.
- [ ] Request loop handles re-login, nonces/CSRF tokens, required sequences.
- [ ] Throttling configured; lockout/anomaly pause logic in place.
- [ ] All responses logged for post-hoc re-analysis by each signal axis.
- [ ] CAPTCHA enforcement (not presence) verified before automating past it.
