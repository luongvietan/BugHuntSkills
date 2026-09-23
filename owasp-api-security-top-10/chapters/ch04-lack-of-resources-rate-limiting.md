# API4:2019 — Lack of Resources & Rate Limiting

> **2023 status:** broadened → **API4:2023 Unrestricted Resource Consumption**
> (response size, CPU/memory, quotas, per-request cost — not just request rate).
> All testing stays low-volume: prove the mechanism, never degrade service.

## Core Idea
APIs without request/size/record limits can be exhausted by simple repeated calls — no auth needed, single machine or cloud. Enables DoS directly and unlocks brute-force attacks on auth/OTP endpoints.

**Scores**: Exploitability 2 · Prevalence 3 · Detectability 3 · Technical Impact 2

## Is the API Vulnerable? (Test Conditions)

A missing or mis-set limit on **any** of these makes it vulnerable:

- Execution timeouts
- Max allocable memory
- File descriptors / process count
- **Request payload size** (uploads)
- **Requests per client/resource per timeframe**
- **Records per page in a single response**

Resource cost per request scales with *user input* and endpoint logic — find the input that maximizes server work.

## Example Attack Scenarios

**Scenario 1 — thumbnail bomb**: `POST /api/v1/images` accepts a huge image; server-side thumbnail generation exhausts memory → API unresponsive.

**Scenario 2 — page-size amplification**: UI lists users via `/api/users?page=1&size=200`; attacker sets `size=200000` → DB strain → DoS. Same pattern can trigger **integer/buffer overflows** — also probe oversized values for crash-type bugs, not just slowdown.

## How To Prevent

- Container-level limits (Docker: memory, CPU, restarts, fds, processes).
- Client rate limiting per timeframe; return limit value + reset time on exceed.
- Server-side validation on query/body params — especially record-count controls.
- Enforce max data size on all incoming params (string length, array element count).

## Anti-patterns

- **Rate limit only at the gateway**: check it actually applies to the tested path/method/host (see API9 beta-host gap).
- **No cap on `size`/`limit`/`page` params** — classic amplification.
- **Per-IP only**: test whether the limiter keys on account, token, or IP — rotation bypasses naive IP limits (report carefully, stay in scope).

## Key Takeaways

1. Probe numeric params (`size`, `limit`, `per_page`, `count`) with extreme values — watch latency/errors.
2. Upload endpoints: oversized file + server-side processing = resource exhaustion.
3. Missing rate limits on auth/OTP = brute-force enabler (API2's attack surface).
4. DoS findings in bug bounty need care: document the *mechanism* (a few requests), never actually take the service down.
5. Also test timeouts on expensive queries — long-running requests hold resources.

## Connects To

- **API2 (ch02)**: OTP/reset brute force exists *because* of missing rate limits
- **API9 (ch09)**: beta/old hosts often lack the gateway's rate limiting
- CWE-307, CWE-770; NIST rate-limiting guidance
