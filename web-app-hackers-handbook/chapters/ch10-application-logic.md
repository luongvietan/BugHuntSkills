# Ch11: Attacking Application Logic

Source: Chapter 11. No signature, no scanner — logic flaws are **defective assumptions**. The developer reasoned "if A happens then B, so do C" and never asked "what if X?" They survive code review and pentests because finding them requires understanding the workflow, then violating its assumptions. Highest-value, least-automatable category.

## The assumption taxonomy (12 canonical patterns)

| Pattern | Assumption | Attack |
|---|---|---|
| Encryption oracle | one crypto use is isolated | feed *other* encrypted values to any decrypt/re-encrypt point (ex: swap `ScreenName` cookie for `RememberMe`; set screen name to `admin|1|ip` → forge admin token) |
| Absent-parameter switch | presence/absence of a param distinguishes roles | **remove** the param (name+value — not empty string) → admin code path (password-change ex) |
| Stage order | users follow the UI sequence | forced browsing: skip payment stage → free order; replay/duplicate stages |
| Shared param processor | requests carry only expected params | submit *any* stage's params at *any* stage → validation skipped; submit another role's params → self-approval |
| Session-state pollution | session objects used per feature | populate identity-bearing session object via *another* feature (registration inside authenticated session → become the registered customer) |
| Numeric limits | threshold check covers the range | negative amounts bypass `<= threshold` checks (`-$20,000` transfer = reverse direction, no approval) |
| One-time adjustments | adjustment computed once stays valid | earn discount on full cart, then remove items → discount retained |
| Escaping | escaping metachars neutralizes input | escape the escape: `foo\;ls` — `\` escapes the backslash, `;` executes |
| Validation ordering | each step sees final data | truncation after quote-doubling re-opens injection; non-recursive strips (`<scr<script>ipt>`); decode-after-check |
| Search index scope | results only show what you may see | match *counts* are an oracle — iterative queries infer protected doc contents (letter-by-letter password extraction) |
| Debug/error output | messages only reach their owner | error info in *static* storage → poll the error URL → harvest other users' tokens, params, passwords |
| Thread safety | per-user data stays per-user | static/shared variable for session data → race window → login lands in another user's session |

## Attack method

1. **Map workflows end-to-end** and write down every assumption ("users do X before Y", "only admins send param Z", "values stay positive").
2. **Remove parameters** one at a time (delete the name too — absent ≠ empty); each removal probes a code path.
3. **Out-of-sequence requests**: skip stages, replay stages, access later stages early, earlier stages late — watch for null/partial state and interesting errors.
4. **Cross-submit parameters**: params from stage A at stage B; params from admin's flow in a user's request.
5. **Accumulate state then switch context**: perform feature 1 (which writes session state), then feature 2 and observe.
6. **Numeric edge cases**: negative, zero, huge (overflow), decimals where ints expected.
7. **Adjust-then-mutate**: trigger a computed value (price/discount/quota), then change its inputs.
8. **Race conditions** (specialized): pick functions that read-then-write shared state (login, transfer, single-use token); script many parallel users/actions from several connections; expect low yield and false positives — a genuine anomaly is reproducible statistically, not once.
9. **Search/count oracles**: use match counts and inference to extract data the UI withholds.

## Checklist

- [ ] Every workflow's assumptions listed; each violated at least once.
- [ ] Param-removal sweep on all key functions (login, password change, transfer, admin).
- [ ] Stage-skip/replay sweep on all multistage processes — followed through to completion.
- [ ] Cross-role parameter submission tried wherever roles share functionality.
- [ ] Negative/overflow values in every numeric business limit.
- [ ] Crypto oracles: every decrypt/encrypt point fed foreign values.
- [ ] Error/debug endpoints probed for cross-user data (two-user parallel testing).
- [ ] One-time-use items (tokens, discounts, limits) retried concurrently.
