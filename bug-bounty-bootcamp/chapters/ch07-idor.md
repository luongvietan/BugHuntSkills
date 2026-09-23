# Ch10: Insecure Direct Object References

Source: Chapter 10. IDOR = the app exposes a reference to an object (ID in URL/body/filename) and fails to check whether *this* user may access *that* object. Li's framing: it's **broken object-level authorization**, not merely "guessable IDs" — unpredictable UUIDs don't fix missing authZ.

## Hunting

1. **Two accounts, every role.** Create A (victim) and B (attacker). Capture A's requests that fetch/modify A's objects, replay with B's session — read works? modify works?
2. **Find the identifiers**: numeric IDs (`/users/123`), UUIDs, emails, usernames, filenames (`/files/report_2021.pdf`), encoded IDs (base64 of `user:123`), composite keys.
3. **Add parameters that weren't there**: the request has `username=me` — try `&user_id=`, `&id=`, `&account=`, `&message_id=` — apps often honor hidden params (mass-assignment cousin).
4. **Both directions**: read IDOR (data leak) and write IDOR (modify/delete others' objects — higher severity).
5. **Unpredictable IDs aren't safe**: leak them. Public profiles, shared URLs, "export" features, invite links, API responses that return friend/follower IDs — the book's case: endpoint leaks user + all friends' IDs → read private messages between them.
6. **Blind IDOR**: the response doesn't show data, but a side effect does — email triggered to the victim, export file generated, notification sent, receipt emailed, async job queued. Check your mailbox/exports.

## Bypassing protections

- Encoded/hashed IDs → decode (base64), or find the leak that maps hash→ID.
- "UUID-only" checks → find any numeric-ID alias endpoint, or old API version (`/v1/` vs `/v2/`).
- Validation on GET but not POST (or vice versa) → switch methods; try `POST /users/123` vs `GET`.
- Front-end hides the field → request it directly via Burp.

## Escalation & automation

Rate by object sensitivity: messages > PII > settings > public data; write > read. Chains: IDOR → read reset token → ATO; IDOR → modify email → password reset → ATO; IDOR → admin object. Automate with Burp **Autorize/AuthMatrix** (replay as low-priv user, diff responses) and Wfuzz over ID ranges (`wfuzz -w ids.txt "...?user_id=FUZZ"`) — valid IDs show as 200/longer responses.

**First-IDOR checklist**: find object references → replay cross-account → add phantom params → hunt ID leaks → check side channels → escalate toward ATO.
