# sources.md — payloads-all-the-things

## Sources

- **Primary**: PayloadsAllTheThings — https://github.com/swisskyrepo/PayloadsAllTheThings
  — community-maintained payload library (~40 vuln-class directories).
  Chapters here are condensed/reorganized, not verbatim.
- Companion for mechanism depth: `web-security-academy` (PortSwigger),
  `xss-cheat-sheet` (XSS-specific depth), `hacking-the-cloud` (SSRF→cloud
  chain).
- Methods-refresh source register: `docs/superpowers/2026-09-23-bug-bounty-source-register.md`
  (BugBounty workspace).

## Payload policy (live programs)

- **Context first, detection first, harmless proof only.** The library
  documents payload *families* including lab-grade exploit chains — on a
  live program the first payload that confirms the class is the last you
  send (`alert(document.domain)`, a 5s sleep, a DNS hit to your listener,
  a canary file).
- Per-chapter `> **Live-program ceiling:**` callouts define the stop line
  per class; escalation payloads are report narrative, not procedures.
- OOB callbacks only to tester-controlled or program-authorized
  infrastructure; volumetric/DoS-class payloads are lab-only.

## Currency

- Payloads drift slower than tools but filters move fast — WAF/framework
  bypass sections are point-in-time; mutate for the *observed* filter,
  not the documented one.

## Review history

- 2026-09-23: live-program ceiling callouts on 8 chapters, lab-vs-live
  doctrine line in SKILL.md, sources.md created (methods-refresh T8).
