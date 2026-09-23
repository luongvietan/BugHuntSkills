# 01-phase-map.md — the seven phases, routed

One section per phase: purpose -> inputs -> concrete actions -> which skill and
chapter files to load -> exit criteria -> do-not-skip warnings. Paths are
relative to the skills root (`~/.agents/skills/`); every file listed exists on
disk — verify with `ls <skill>/chapters/` if a path ever 404s after an update.

Phases are a loop, not a ladder: phase 7 feeds phase 2, and any phase can send
you back a step (a mapping surprise restarts recon on one host; a report
question sends you back to phase 5 for a better PoC).

---

## Phase 1 — program selection

**Purpose:** pick a program where your skills, time budget, and the
competition level line up — and read the policy before touching anything.

**Inputs:** platform listings (HackerOne/Bugcrowd/Intigriti/self-hosted VDP),
your hours-per-week, prior reports.

**Actions:**

- Read the full policy end-to-end and pin it: record the policy URL, the
  revision or the date you read it, in-scope asset list, exclusions, banned
  techniques (DoS, social engineering, automated scanning limits, brute
  force), safe-harbor wording, rate limits, payout table, response SLAs, and
  the stop/contact conditions. Write it all down — see
  `02-session-checklist.md` for the exact contract.
- Produce two artifacts, not one mental note: `hunt/<target>/scope.md` (the
  human contract) and `hunt/<target>/allowlist.txt` (the machine-readable
  list every active stage draws targets from — recon-pipeline consumes it).
- Check signal-vs-noise: recent resolved reports, payout ranges, triage speed,
  program age (new programs = fresh surface; old ones = picked-over but deeper
  bugs left).
- Prefer programs matching your strongest vuln classes and tech stack; two
  accounts' worth of functionality you can fully exercise beats a famous
  target you can only read.

**Skill routing:**

- `tbhm-methodology/chapters/ch01-philosophy-bounty-model.md` — how the bounty
  model shapes what gets paid; pick targets the model rewards.
- `zseano-methodology/chapters/ch01-mindset-program-notes.md` — program notes
  template, mindset, picking what to work.
- `bug-bounty-bootcamp/chapters/ch01-industry-reports.md` — industry layout,
  what reports look like end-to-end before you commit.
- `web-hacking-101/chapters/ch01-methodology.md` — compact methodology
  overview if you want the short version first.

**Exit criteria:** policy read; scope includes/excludes written to
`hunt/<target>/scope.md`; banned techniques noted; two test accounts planned.

**Do not skip:** the exclusions list. Hunters lose accounts to technically
in-scope-looking assets (CDNs, acquired-company domains, third-party SaaS)
that the policy carves out in one sentence.

---

## Phase 2 — recon

**Purpose:** enumerate every in-scope asset — subdomains, live hosts, ports,
directories, screenshots, code leaks, tech fingerprints — normalized so the
next run can be diffed.

**Inputs:** `hunt/<target>/scope.md` from phase 1; previous run's output if any.

**Actions:**

- Stage 0 is the allowlist: `hunt/<target>/allowlist.txt` is the only source
  any target-touching stage may draw from. No allowlist file or an empty one
  → the active target list is empty — passive collection only, or nothing.
- Passive collection first (third-party sources: CT logs, search engines,
  archives, DNS zone data, repo/search-code surfaces) — sends no packets to
  the target. Everything it emits is a *lead*.
- Target-traffic stages (HTTP probing, DNS queries against target
  infrastructure, screenshots) run only on allowlist-derived targets after
  the authorization re-check.
- Intrusive/high-volume stages (port scanning, directory brute-force, DNS
  brute-force) are a separate gate again — explicit policy permission plus
  conservative rate; never the default.
- Land every artifact in `recon/<target>/<YYYYMMDD>/<stage>.txt`; normalize
  (`sort -u`, lowercase, one asset per line); diff vs prior run into
  `new-since-last-run.txt`. Discovered assets stay leads until the ownership
  + allowlist re-check promotes them.
- Tech fingerprint + asset types feed phase 4: an API-heavy stack routes you
  to `hacking-apis`; cloud metadata/buckets route to `hacking-the-cloud`.

**Skill routing:**

- `recon-pipeline/chapters/01-stages.md` — the seven stages with verbatim
  commands and per-stage passive/active gating.
- `recon-pipeline/chapters/02-outputs.md` — dated layout, normalization,
  `comm`/`diff` vs prior run, lead-file format.
- `tbhm-methodology/chapters/ch02-discovery-recon.md` — discovery concepts
  behind the commands (vertical vs horizontal enumeration, ASN/cert tricks).
- `bug-bounty-bootcamp/chapters/ch03-recon.md` — Li's recon chapter: subdomain
  enum, certs, GitHub dorking, S3 buckets, directory brute-force, automation.
- `zseano-methodology/chapters/ch02-basic-toolkit.md` — lean toolkit if the
  full pipeline is overkill for this program.
- `hacking-the-cloud/chapters/ch01-recon-unauthenticated-enum.md` — when the
  scope includes cloud orgs/tenants: unauthenticated enumeration of
  subscriptions, buckets, principals.

**Exit criteria:** dated run dir exists; `new-since-last-run.txt` written;
live-host list pushed to Burp scope (allowlist-derived only); tech list
annotated with candidate vuln classes.

**Do not skip — the deny-by-default rule:** a discovered hostname or resolved
IP is a lead until ownership AND allowlist membership are confirmed. Shared/
CDN infrastructure (Cloudflare, Fastly, Akamai, third-party SaaS IPs) never
inherits authorization from a related hostname — probe by hostname, never
scan the shared IP. Ambiguous ownership, missing permission, or unexpected
real-user data → halt the stage, re-check policy or contact the program.

---

## Phase 3 — application mapping

**Purpose:** turn "hosts exist" into "here is every feature, role, parameter,
and trust boundary" — the map that phase 4 hunts on.

**Inputs:** live hosts + tech fingerprints from phase 2; test accounts at two
privilege levels.

**Actions:**

- Walk every feature manually at every privilege level; let Burp build the
  site map. Log endpoints, parameters, roles, and state-changing actions.
- Note hidden/undocumented surface: JS-file endpoints, API routes referenced
  by the frontend, alternate subdomains running old versions.
- Classify each function: authZ boundary? parser? file handling? state
  machine? — each answer names candidate vuln classes for phase 4.

**Skill routing:**

- `web-app-hackers-handbook/chapters/ch03-mapping-application.md` — the
  deepest treatment: entry points, hidden content, fingerprinting.
- `tbhm-methodology/chapters/ch03-mapping-enumeration.md` — mapping folded
  into the bounty workflow.
- `owasp-wstg/chapters/ch01-information-gathering.md` — checklist-driven info
  gathering (search engines, comments, metadata, entry-point enumeration).
- `zseano-methodology/chapters/ch05-step-one-first-look.md` +
  `chapters/ch06-step-two-attack-surface.md` — first-look triage and attack-
  surface enumeration in one sitting.
- API-heavy target: `hacking-apis/chapters/ch03-discovering-apis.md` +
  `chapters/ch04-endpoint-analysis.md`.
- Mobile target: `owasp-mas/chapters/ch01-methodology-setup.md` for
  environment + app analysis setup.

**Exit criteria:** endpoint+parameter inventory exists in notes; every
in-scope host walked once; candidate vuln classes listed per function.

**Do not skip:** privilege-level diffing during mapping, not after. Replaying
admin-account requests as the low-priv account is where half of phase 4's
IDOR/BFLA findings actually surface — log both sessions' requests as you map.

---

## Phase 4 — vuln hunting

**Purpose:** work the candidate classes from phase 3, one at a time, with the
per-class chapter open — not a memory of it.

**Inputs:** the map from phase 3; `chapters/03-vuln-class-index.md` for
per-class routing.

**Actions:**

- Pick classes by the map: forms/reflection -> XSS; parameters -> injection
  family; object IDs -> IDOR/BOLA; integrations/webhooks -> SSRF; front-end
  state -> DOM classes.
- For each class: open the routed chapter, run its hunting loop, try its
  documented bypasses against observed defenses, log every anomaly (even
  unexplained ones — they are phase-5 material).
- Payload choice is its own lookup: `payloads-all-the-things` per class, and
  `xss-cheat-sheet` when XSS filters fight back.

**Skill routing (the short version — full table in `03-vuln-class-index.md`):**

- Core web classes -> `bug-bounty-bootcamp` ch04-ch18 (per-class loop:
  mechanism -> hunting -> bypass -> escalate).
- Modern classes -> `web-security-academy` ch01-ch09 (smuggling, HTTP/2
  desync, cache/host, prototype pollution, DOM, WebSockets, JWT, OAuth/SAML,
  CORS/CSRF/clickjacking).
- Checklist coverage -> `owasp-wstg` ch02-ch12 by test category.
- Deep mechanism/evasion -> `web-app-hackers-handbook` ch05-ch14.
- API target -> `hacking-apis` ch04-ch11 + `owasp-api-security-top-10`
  ch01-ch10 as the class list.
- Mobile -> `owasp-mas` ch02-ch08 + `bug-bounty-bootcamp`
  `chapters/ch20-android.md`.
- Exploitation ops (Burp brute-force, CMS checks, cache attacks, OSRF) ->
  `bug-bounty-playbook` ch01-ch14.
- Fuzzing pass after manual -> `bug-bounty-bootcamp`
  `chapters/ch22-fuzzing.md`, `hacking-apis/chapters/ch06-fuzzing.md`,
  `web-app-hackers-handbook/chapters/ch13-automating-attacks.md`.

**Exit criteria:** every candidate class attempted with its chapter's loop;
each candidate bug has a confirming request pair logged in notes; anomalies
list written.

**Do not skip:** the policy's banned-technique list applies hardest here —
rate-limit/volume tests, DoS-shaped payloads, and spam vectors often need
explicit permission. Test accounts only; never pull real user data "to prove
impact" — that proof is built in phase 5 within the policy, not outside it.

---

## Phase 5 — escalation and chaining

**Purpose:** convert "the bug exists" into "a malicious actor gains X" —
escalate single bugs and chain lows into a payable whole.

**Inputs:** confirmed bugs + anomalies from phase 4; program severity table.

**Actions:**

- For each bug, ask the escalation question: what does the next step buy?
  Reflected XSS -> session theft -> ATO? SSRF -> cloud metadata -> keys ->
  control plane? IDOR -> how many objects, whose data?
- Write down chain hypotheses combining your lows (self-XSS + login CSRF +
  CORS misconfig; open redirect + OAuth flow; info leak + authN weakness) and
  test them on your own accounts.
- Re-check impact claims against demonstrated scale only — what you proved on
  test accounts, not what could theoretically exist.

**Skill routing:**

- `bug-bounty-playbook/` — exploitation-phase ops:
  `chapters/ch01-known-vulnerabilities.md`, `chapters/ch05-brute-force-burp.md`,
  `chapters/ch11-cache-attacks.md`, `chapters/ch13-osrf-prototype-csti.md`,
  `chapters/ch14-xxe-csp-rpo.md`.
- `bug-bounty-bootcamp/chapters/ch14-logic-access-control.md` — escalation
  thinking for logic/access-control bugs; each bootcamp vuln chapter ends
  with an escalation section too.
- `web-hacking-101/` — case-study precedent for chains that paid:
  `chapters/ch04-application-logic.md`, `ch07-subdomain-takeover.md`,
  `ch09-rce-template-ssrf.md`.
- `web-app-hackers-handbook/chapters/ch20-methodology.md` — the mature
  attacker's checklist for squeezing a target.
- Cloud escalation -> `hacking-the-cloud/chapters/ch05-privesc-misconfigured-policies.md`
  + `chapters/ch06-lateral-movement.md` when found IAM material or SSRF
  reaches the control plane.

**Exit criteria:** each bug's impact statement is demonstrated, not asserted;
viable chains tested on owned accounts; severity hypothesis per bug.

**Do not skip — minimal viable PoC:** escalate exactly far enough to prove
the claimed impact, then stop. Pivoting further "to show how bad it could be"
reads as exceeding authorization to triage and can breach the safe-harbor
you were operating under.

---

## Phase 6 — reporting

**Purpose:** turn the demonstrated impact into a report a triager can verify
in under five minutes — while the context is still in your head.

**Inputs:** per-bug evidence (requests, responses, screenshots) from phases
4-5; program severity table and report format.

**Actions:**

- One bug per report (a chain is one report *as a chain*). Fill the skeleton:
  title naming impact -> summary -> severity -> description -> numbered repro
  from clean state -> impact -> remediation.
- Write repro steps first — they force you to re-verify the bug before a
  triager finds out for you.
- Attach evidence inline at the step it proves; all identifiers belong to
  your test accounts.

**Skill routing:**

- `report-writing/chapters/01-severity.md` — pick and defend a severity.
- `report-writing/chapters/02-template.md` — skeleton + repro-step rules.
- `report-writing/chapters/03-impact-library.md` — per-class impact phrasing.
- `report-writing/chapters/04-triage.md` — N/A / informative / duplicate /
  dispute handling.
- `web-hacking-101/chapters/ch10-memory-reporting.md` — what paid reports
  looked like in practice.
- `bug-bounty-bootcamp/chapters/ch01-industry-reports.md` — platform/report
  expectations.

**Exit criteria:** repro replays cold in <5 min on named test accounts; impact
section shows attacker outcome with demonstrated scale; scope re-verified;
submitted via the platform channel.

**Do not skip:** platform-channel-only. Nothing about the finding leaves the
report thread until the policy's disclosure terms allow it — no writeup, no
teaser, no "advisory" repo.

---

## Phase 7 — continuous monitoring

**Purpose:** make recon compound — schedule the pipeline, alert on diffs, and
let new-surface notifications time your next hunt while reports are in triage.

**Inputs:** prior `recon/<target>/` runs; `new-since-last-run.txt`; program
changelog/scope-update feed.

**Actions:**

- Schedule the recon pipeline (cron / Task Scheduler); alert on diff output,
  not on run completion.
- Subscribe to program updates — scope expansions are fresh, unhunted surface
  and the single best moment to run a diff.
- Hunt `new-since-last-run.txt` first on return: new subdomains, new ports,
  new endpoints are the least-hunted surface on any program.
- Feed findings back: note which assets/classes produced bugs — phase 1 data
  for the next session's target choice.

**Skill routing:**

- `recon-pipeline/chapters/03-monitoring.md` — scheduling, diff alerting,
  cadence, scope-drift cautions.
- `hacking-the-cloud/chapters/ch07-exfiltration-detection.md` — what the blue
  team sees: which of your techniques trip detections; OPSEC calibration for
  repeat visits.
- `zseano-methodology/chapters/ch07-step-three-findings-resources.md` —
  review findings, feed lessons into the next round.
- `tbhm-methodology/chapters/ch08-mobile-aux-checklist.md` — auxiliary/mobile
  checklist for programs with app surfaces you revisit periodically.

**Exit criteria:** pipeline scheduled; alert fires on diffs; next session's
lead file will exist without a manual kickoff.

**Do not skip — scope drift:** a host that appears between runs (new
acquisition, new subdomain) is a scope question before it is a target.
Re-run the phase-2 ownership/exclusion check on every new-since-last-run
entry before any active follow-up.
