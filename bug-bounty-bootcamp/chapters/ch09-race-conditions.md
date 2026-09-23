# Ch12: Race Conditions

Source: Chapter 12. Race condition = the app checks a condition (balance, coupon use, invite limit) and *then* acts on it — two+ requests arriving in the check→act window both pass. Classic TOCTOU bug.

## Hunting

Look for **limited-use or balance-checked actions**:

- Money/credit transfers, withdrawals, refunds
- One-time coupons, promo codes, discount application
- Invite slots, team-seat limits, resource quotas
- Vote/like/follow counts (impact ceiling, but easy first find)
- Password-reset / OTP validation attempts

Test: send the same request N times *simultaneously*. Tools:

- **Burp Repeater tab groups + "Send group in parallel"** (single-packet attack, HTTP/2) — the modern way; removes network jitter so requests land in the same ms window.
- **Intruder null payloads** with high thread count — works but slower.
- **Turbo Intruder** (`race.py` template) for fine control.
- Curl loop `for i in {1..20}; do curl … & done` — crude but illustrative.

Success signal: coupon applied twice, balance goes negative, 6 seats in a 5-seat plan, transfer exceeds balance.

## Why it happens & escalation

App does `SELECT balance` → checks `balance >= amount` → `UPDATE balance` in separate queries with no locking/transaction. Anywhere state is read-then-written is a candidate: inventory, reservation systems, file uploads with name dedup, account-merge flows.

Impact story for the report: quantify ("redeemed the $100 coupon 14× = $1400 fraud per code"). Escalate by combining with an endpoint that *creates* the limited resource (e.g., race the refund AND the purchase).

## First-bug checklist

List limited-use actions → capture the request → parallel-send 10-20× → check for doubled effect → document the race window and per-run gain → note that fixing = atomic transaction/DB lock, not rate limiting.
