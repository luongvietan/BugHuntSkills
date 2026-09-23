# Triage Playbook — Verdicts, Disputes, Mediation

The verdict isn't the end of the report — it's the start of a negotiation you mostly can't win and occasionally can. The only currency that moves a verdict is **new evidence**. Tone, persistence, and reputation decide whether that evidence gets a fair read.

## Verdict decoder

| Verdict | What it means | What usually caused it | Is it worth disputing? |
|---|---|---|---|
| **Not applicable (N/A)** | "No security impact" or "not a vuln" or out of scope/policy | Impact section described the bug, not the attacker outcome; or asset/technique genuinely out of scope | Only if you can attach *new* impact evidence — a chain step, a second affected object, a missed precondition. Repetition loses. |
| **Informative** | Real issue, real weak impact — "thanks, no bounty" | Bug is real but standalone impact is thin (missing header, verbose error, weak redirect) | Rarely. Either escalate it into a chain and resubmit, or take the signal: this program wants demonstrated harm. |
| **Duplicate** | Someone reported it first | You hunt where everyone hunts | Almost never wins bounty. Ask *once* whether it was a known issue vs. a recent submission — then treat it as recon data. |
| **Spam** | Report flagged as noise/invalid submission | Automated submission, wrong program, unverifiable claim, or pattern of low-effort reports | Serious — it dings your reputation/signal. Only dispute if clearly mistaken; otherwise stop, review your last reports for quality drift. |
| **Out of scope** | Asset or technique excluded | Policy mismatch — domain not listed, banned technique (DoS, social engineering) | Only if the scope text genuinely covers it (quote the line). "But it's the same app" loses. |
| **Triaged → severity lower than claimed** | Accepted, scored down | Over-scoped CVSS metrics; scale claimed but not shown | Sometimes — see the argue-up table in `chapters/01-severity.md`. Accept with grace unless you hold concrete new evidence. |
| **Resolved + bounty** | Done | — | Thank-you note naming the fix helps; verify the fix later if policy allows retest. |

## Response playbook per verdict

### N/A — the one real comeback

1. Re-read your report as the triager saw it: is the attacker outcome *demonstrated* anywhere, or only implied?
2. Build the missing piece: the chain link you skipped, the sensitive field you didn't show, the pre-auth variant you didn't test. This must be **new work**, not rephrasing.
3. One reply, structured: acknowledgment → new evidence (numbered steps, attached PoC) → the severity band it now matches. No argument about the first verdict — litigate the evidence, not the decision.
4. If still N/A after the best evidence you can produce: the disagreement is honest. Take mediation only if the evidence is airtight (below) — otherwise walk.

Template spine (adapt, don't paste verbatim):

> Thanks for the review. I've added a demonstration that closes the impact gap: [1-line claim]. New steps below show [victim outcome] on my own test accounts — this wasn't in the original report. Given [X], I believe this matches the program's [band] definition: "[quote the row]". Happy to run any specific check you'd like to see.

### Informative — accept or upgrade, don't argue

An informative verdict is usually correct *for what you submitted*. Two paths: (a) the bug chains — build the chain and submit the *combined* finding as a new report referencing the old one; (b) it doesn't — bank it as a program-knowledge data point and move on. Replying "but this is still a real bug" converts a neutral verdict into a negative impression.

### Duplicate — extract value, exit fast

One polite question is acceptable: *"Understood — was this a known issue, or recently reported by another researcher?"* The answer tells you whether the program sits on bugs (long-lived known issue → deprioritize this target class) or whether you're sharing recon space with the crowd (recent report → differentiate your hunting ground). Never argue you found it independently — duplicates pay the first reporter, full stop.

### Spam — treat as a quality alarm

One spam verdict = investigate yourself before disputing: was the claim verifiable? Was the asset right? Did the report read like a template blast? If it's genuinely mistaken (e.g., automated filter ate a legit report), one calm message to support/the program pointing to the specific evidence. If your last N reports are thin, the fix is upstream — tighten scope, strengthen PoCs — not in the dispute box.

## Dispute etiquette — the rules that protect your account

- **One evidence-bearing reply, then stop.** Every additional message after your best evidence reads as pressure, not information. Triagers escalate patterns of pressure.
- **Argue impact, never money.** "This deserves High because [X users, Y data]" works; "Program Z paid $N for this" poisons the thread. Bounty tables are the program's call.
- **Quote their own policy.** Scope lines, severity tables, safe-harbor text — the strongest disputes are the program's words applied to your evidence.
- **Stay technical in the thread.** No frustration, no venting, no references to time invested. The thread is permanent; future triagers read it.
- **Never reopen closed reports to argue.** If new evidence surfaces, reply once in-thread (platforms allow it) or submit a new report cross-referencing — don't spam status changes.
- **No public pressure.** Naming the program on social media mid-dispute is a policy violation on most platforms and turns a winnable dispute into an account problem.

## Mediation — when and how

Both HackerOne and Bugcrowd run formal mediation; smaller platforms have informal equivalents (support ticket → program owner review).

**Use it when ALL of these hold:**
- The evidence is objective and reproducible — a third party can run your steps and see the impact.
- The disagreement is about *application of policy* (severity band, scope interpretation), not about whether the bug exists.
- You've already made your single best-evidence reply and it was declined or ignored past the program's stated SLA.
- You can afford the relationship cost — mediation is a legitimate channel, but programs remember who invokes it. Spend it on findings that matter (critical/high classed as N/A), not on mediums.

**Do NOT mediate:** duplicates (never reversed), genuine scope exclusions, "I deserved more bounty" (amounts are rarely mediated), or cases where your evidence is "trust me, it's exploitable".

**If you go:** file with documentation — the report link, your evidence reply, the policy text you're invoking, and a timeline. Write the mediation request like a report: claim → evidence → policy line → requested outcome. Then go quiet; adding commentary mid-mediation weakens it.

## Timelines that keep you sane

- Programs have published SLAs (time-to-triage, time-to-bounty). Nudge **once** after SLA breach; a second nudge adds nothing.
- Silence ≠ rejection. Use waiting time on other targets; a report you're refreshing daily pays the same as one you're not.
- After a verdict, wait ≥24h before replying unless new evidence is ready. Fast emotional replies are how reputations die.

## When to walk away

- You've delivered your best evidence and the verdict stands — further arguing trades future-report credibility for this one.
- The program's pattern is hostile triage across *many* reporters (check platform stats/disclosures) — your reports are worth more elsewhere; programs that won't pay aren't a character flaw to fix, they're a lead-quality problem.
- The dispute would require revealing more exploitation than policy allows (to "prove" impact you'd have to touch real user data) — policy wins, withdraw gracefully, keep the account.
- It's the third dispute this quarter on the same program — you're either hunting in their blind spot (adjust targeting) or misreading their taxonomy (recalibrate with `chapters/01-severity.md`). Either way, the next report matters more than this fight.

## The long game

Reputation is compounding infrastructure: triagers triage *reporters*, not just reports. A hunter who accepts verdicts gracefully, disputes rarely but with evidence, and never litigates money gets faster reads, benefit-of-the-doubt on borderline severity, and private-program invitations. Protect that asset harder than any single bounty.
