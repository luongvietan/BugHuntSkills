# Report Template — Skeleton + Worked Example

Copy the skeleton, fill every section, delete guidance in `*(italic)*`. A triager should reproduce from clean state in **under 5 minutes** without asking a question — that constraint shapes every section.

## Title format

```
[class] in [feature] at [endpoint] → [attacker-visible impact]
```

- Name the **impact**, not just the bug: `IDOR in order history at /api/orders → read any customer's invoices` beats `IDOR on order endpoint`.
- Keep it under ~15 words; the title is the triage queue's summary.
- Include the path — it's how duplicates get spotted and how the team routes it.

## Skeleton

```markdown
## Summary
*(2–3 sentences: vuln class + affected component + business impact. Write it
last, when you know exactly what you proved.)*

## Severity
*([Band] — e.g., High. CVSS vector + score if the platform scores; otherwise
one line tying the band to the program's published severity table.)*

## Description
*(1 paragraph technical mechanism: what's missing/wrong. Precision here —
"the endpoint authorizes the caller's role but never checks ownership of the
`order_id` object" — is what makes the repro believable.)*

## Steps to Reproduce
*(Numbered. Start from a clean state: fresh incognito session, named test
accounts. Include exact requests the triager can replay verbatim.)*

1. Register two test accounts: `attacker@test.example` and `victim@test.example`.
2. *(Setup step — log in, create the state the bug needs.)*
3. *(The request — paste the full raw request, or a curl equivalent with
   placeholders only for the session token.)*
4. *(Expected vs. actual — "Expected: 403. Actual: 200 with victim's data.")*
*(Attach a screenshot inline at the step that produces the evidence.)*

## Impact
*(What an attacker gains, quantified: data class × scale × privilege.
Pull phrasing from impact-library.md; claim only what the PoC showed.)*

## Remediation
*(Concrete fix: "enforce object-level authorization on `order_id` against the
caller's account ID" — not "implement proper access control". 1–3 bullets.)*

## Environment *(optional but cheap)*
*(Date tested, endpoint host, account roles used, tool versions if relevant.)*
```

## Section rules (where reports actually lose triagers)

**Summary** — if the triager reads only this, do they know class + component + impact? No background, no "I found during testing…".

**Severity** — one band + evidence. If you list a CVSS vector, every metric must be defensible from the repro steps below (see `severity.md`).

**Steps** — the make-or-break section:
- Step 1 is always environment setup: accounts, roles, starting URL. Never assume the triager's session state.
- Paste **raw requests** (method, path, headers that matter, body) — screenshots of Burp are supplements, not substitutes.
- Number every step; one action per step; state expected vs. actual at the exploit step.
- Re-verify by replaying your own steps in a fresh session before submitting. If it takes you 8 minutes, the triager needs 15 — tighten it.
- Parameter values from your test accounts only; redact nothing the triager needs (they can't repro `order_id=REDACTED`).

**Impact** — attacker-outcome language: reads what, changes what, costs whom. Quantify: "all ~enumerable order IDs", "any user with a public profile". Never speculate in this section — chains go in with evidence or in a labeled "possible escalation" sentence.

**Remediation** — match the mechanism: missing object check → object-level authZ; string-built query → parametrized statements; missing token → synchronizer token bound to session. Vague fixes ("sanitize inputs") signal you didn't understand your own bug.

**Attachments** — screenshots at the step they prove, cropped to the evidence. Video only for timing/multi-step bugs; still include the text steps.

## Worked example — IDOR

```markdown
Title: IDOR in order export at /api/v1/orders/export → download any
customer's invoices (PII + payment details)

## Summary
The order-export endpoint authorizes the caller's session but never verifies
that the requested `order_id` belongs to that account. Any authenticated user
can iterate sequential order IDs and download other customers' invoices,
which contain names, addresses, and the last four digits of payment cards.

## Severity
High — CVSS:3.1/AV:N/AC:L/PR:L/UI:N/S:U/C:H/I:N/A:N (6.5), upgraded one band
in argument below because the exposed fields include payment data on every
historical order, not a limited subset. Aligns with the program's "High:
sensitive data exposure at scale" band.

## Description
`GET /api/v1/orders/export` accepts an `order_id` path parameter and returns
a PDF invoice. The handler checks only that a session exists; there is no
join between `order_id` and `account_id`, so any valid ID in the sequential
space is downloadable.

## Steps to Reproduce
1. In a fresh incognito session, register `bb-attacker@test.example`
   (account A) and, in a second browser, `bb-victim@test.example` (account B).
2. As account B, place a test order (SKU "TEST-1", $1.00 test payment). Note
   the assigned `order_id` from the confirmation URL — in this run: 90412.
3. As account A, replay this request:

   GET /api/v1/orders/export/90412 HTTP/1.1
   Host: shop.example.com
   Cookie: session=<account-A-session>

4. Expected: 403 or 404, since order 90412 belongs to account B.
   Actual: 200, `application/pdf` — the invoice renders account B's name,
   billing address, and card brand/last-4. (Screenshot attached: step4.pdf)
5. Repeated with IDs 90410 and 90411 — both returned invoices belonging to
   two other test orders I created under account B. No rate limiting
   observed on this endpoint.

## Impact
An authenticated attacker can enumerate the sequential `order_id` space and
retrieve every customer's invoice: full name, billing address, order
contents, and card brand + last-4. Order IDs issued since launch appear
contiguous (~90k range), so this exposes the full order history at script
speed. No elevated role is needed — a free account suffices.

## Remediation
- Enforce object-level authorization: verify `order.account_id == caller.id`
  before serving the export; return 404 (not 403) to avoid confirming ID
  existence.
- Consider unguessable order references (UUIDs) as defense in depth — not as
  the primary fix.
- Add rate limiting / alerting on bulk export patterns.

## Environment
Tested 2026-09-30 against shop.example.com (in-scope per program policy
v14). Accounts A and B are free-tier test accounts I own; no third-party
order data was accessed beyond the two IDs listed above.
```

## Why this example works

- **Title** = class + feature + endpoint + impact; a triager can route and de-dupe from it alone.
- **Repro** names both accounts, gives one raw request, states expected-vs-actual, and the "victim" data is the reporter's own test account — impact proven without touching a real user.
- **Impact** quantifies (contiguous ~90k range, free account sufficient) and claims only fields actually observed in the PDF.
- **Severity** shows the vector *and* maps to the program's own table — two ways for triage to say yes.
- **Step 5** pre-empts the "did you check rate limiting?" round-trip — the most common source of "needs more info".
