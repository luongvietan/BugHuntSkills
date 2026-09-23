# API10:2019 — Insufficient Logging & Monitoring

## Core Idea
Without logs/alerts, attackers operate undetected — median breach detection is 200+ days, usually by outsiders. For bounty hunters: noisy attack traffic often goes unnoticed; the finding is the *absence* of detection for events that should alarm.

**Scores**: Exploitability 2 · Prevalence 3 · Detectability 1 · Technical Impact 2

## Is the API Vulnerable? (Checklist)

- No logs, wrong log level, or logs missing identifying detail
- **Log integrity unguaranteed** (log injection possible)
- Logs not continuously monitored; no alerting
- API infrastructure unmonitored

## Example Attack Scenarios

**Scenario 1 — unassessable key leak**: admin API keys leaked on a public repo; owner notified but took 48h to act. Insufficient logging → company **cannot determine what data was accessed** — undetectable blast radius.

**Scenario 2 — invisible credential stuffing**: video platform hit by large-scale stuffing; failed logins *were* logged but **no alerts configured**. Attack surfaced only via user complaints → forced mass password reset + regulatory disclosure.

## How To Prevent

- Log all failed auth attempts, denied access, input-validation errors — in a log-management-consumable format with attacker-identifying detail.
- Treat logs as sensitive data; guarantee integrity at rest and in transit.
- Continuous infra/network/API monitoring; SIEM aggregation across the stack.
- Custom dashboards + alerts for suspicious activity.

## Anti-patterns

- **Logging without alerting**: written logs nobody watches = detection time still ~0.
- **Missing actor detail**: logs that can't identify *who* (IP, account, token) are forensically useless.
- **Log injection unchecked**: attacker-controlled input forging/erasing trail.

## Key Takeaways

1. For testing reports: note when obvious attack patterns (mass 401s, stuffing, scans) produce no visible response — evidences missing monitoring.
2. Log-injection check: inject CRLF/`%0d%0a` and format strings into fields that land in logs (User-Agent, names, params).
3. Rate-limit absence + no alerting = brute-force window stays open (chains API2/API4).
4. Impact framing: detection failure turns a small bug into an unbounded breach window.
5. CWE-223, CWE-778; OWASP Logging Cheat Sheet, ASVS V7.

## Connects To

- **API2 (ch02)**: credential stuffing scenario succeeded without triggering a single alert
- **API4 (ch04)**: missing rate limits + missing monitoring = silent brute force
- **API9 (ch09)**: shadow hosts usually also escape monitoring coverage
