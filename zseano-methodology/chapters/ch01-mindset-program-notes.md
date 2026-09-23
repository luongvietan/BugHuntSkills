# Mindset, Choosing a Program & Note-Taking

> **Currency note:** program-choice heuristics are durable (age, scope size, response time); payout norms and platform features are era-marked. Current program/severity policy flows through `bug-bounty-hunter` session init and `report-writing`.

## Core Idea
Bug bounty success = curiosity-driven questioning + months of depth on ONE program. You're not spraying payloads — you're reverse-engineering how developers think, finding their filters, and asking "what did they forget?"

## Frameworks Introduced

- **"Hackers question everything"** — the base mental model. Every feature was coded by a human under deadline; ask what they considered, then test the outcome. Code takes a parameter (GET/POST/JSON), reads the value, executes — probe parameters not referenced anywhere (beta/unused features have weaker protection).
- **The lead system** — first pass isn't about finding bugs, it's about finding *filters*: "Where there's a filter there is usually a bypass." A discovered filter = a lead to chase and a fingerprint of their security posture.
- **"The trend is your friend"** — developers repeat the same mistakes across the codebase and across the industry; one bug class found = hunt it site-wide.
- **Months, not weeks** — pick wide-scope, well-known programs and stay for months. Bigger company → more teams → more mistakes. Learn how their devs think by watching how they patch and what new features ship.

## Choosing a Program

Criteria:
- **Wide scope + recognizable names** — more surface, more teams, more mistakes; different TLDs (.CN) may run different codebases.
- **Known to you** — sites you already use give free mental maps.
- **Private vs public** — privates are less crowded but publics still pay well; VDPs for practice with limits (don't burn out doing free full tests).
- Judge program quality by: direct team communication vs. managed service, activity/scope update recency, how they reward chained low-hanging fruit, response time across ~3-5 reports (>3 months = walk away).
- **Risk vs reward is yours to manage** — "be in the driver's seat"; leave bad programs.

## Writing Notes (the underrated lever)

- Capture *while* browsing: interesting endpoints, behaviors, parameters, what you tried, what you suspect it's vulnerable to — revisit with fresh eyes later; prevents burnout.
- Convert notes into **custom wordlists**: `examplecom-endpoints.txt`, `params.txt` → merge into `global-endpoints.txt` — over time you pre-map any target.
- Gut rule: tired of testing a feature → move on, note it, come back.

## Anti-patterns

- **Trying every vuln type on day one** — burnout and confusion; master a few, expand later.
- **Assuming it works as intended** — always verify; devs code the happy path.
- **Weeks-long commitments** — you can't find all bugs in weeks; trust the process.
- **Demanding payment on VDPs** — that's extortion, not bounty hunting.

## Key Takeaways

1. First pass = filter hunting, not bug hunting. Filters reveal both the bug and the dev's mental model.
2. Unreferenced parameters (`&img=` nobody uses) = beta features with less protection — try everything.
3. Patches are intelligence: how they fix reveals how they think; one fix ≠ site-wide fix.
4. New features interacting with old features = prime business-logic bug territory.
5. Notes→wordlists→custom tools is the compounding asset of this methodology.

## Connects To

- **ch05**: the question lists operationalize this mindset per feature
- **ch06**: notes feed the dorking/wordlist/second-pass work
- **ch07**: case studies show patches/leaks re-exploited via dev-thinking
