# Ch9: API Fuzzing

Source: Chapter 9. Fuzzing = send varied input to provoke unintended behavior. Success depends on **where** you fuzz and **what** you fuzz with.

## Effective fuzzing

Target inputs that reach interesting backends: auth forms, registration, uploads, profile/account editing, user management, search — anything touching a DB, filesystem, OS, or interpreter.

Match the payload to the expected input. Bank transfer `{"userid":12345,"account":224466,"transfer-amount":1337.25}`:
- `transfer-amount`: quadrillion-scale number, letters, huge/negative decimals, `null`/`(null)`/`%00`/`0x00`, symbols `!@#$%^&*();':''|,./?>`
- Same class of probes per field; one field at a time if controls might trip.

Generic input classes that provoke useful errors: oversized numbers/strings, DB queries (`' OR 1=1-- -`), OS commands (`|whoami`), metachars, null bytes, exotic unicode (漢, Ѫ, Ѧ), emojis.

**Escalation path:** generic fuzz → observe error type → targeted fuzz matched to revealed tech (don't send SQLi to a NoSQL app).

**Payload sources:** SecLists `big-list-of-naughty-strings.txt` + Fuzzing dirs, fuzzdb, Wfuzz's `All_attack.txt`, or roll your own (long A-strings, big 9s, symbol soup, `%00`, `$ne`, `%24gt`, `' OR 1=1-- -`, unicode, emoji).

## Detecting anomalies

- Properly handled input → uniform 4xx + tailored error (`{"error":"number required"}`). Improperly handled → 200 + `SQL Error: syntax…`, 500s, timing deltas, verbose dumps.
- **Baseline**: send expected/failing requests first; if 98/100 responses share code+size, that's the baseline — investigate the 2 outliers.
- **Comparer**: send suspect responses to Burp Comparer → Compare Words → Sync Views highlights byte-level diffs in near-identical responses.

## Fuzz wide vs fuzz deep

| | Wide | Deep |
|---|---|---|
| What | One payload across *all* unique requests | Many payloads across every part of *one* request (headers, params, query, path, body) |
| Finds | Improper assets, valid methods, token-handling issues, info disclosures | BOLA, BFLA, injection, mass assignment |
| Tool | Postman Collection Runner | Burp Intruder, Wfuzz |

**Wide with Postman:** create a fuzzing Environment with `{{fuzz1}}`-style variables → insert into token header / URL parts / param placeholders (Find & Replace swaps `<email>`,`<string>` tags across the collection) → add a Tests assertion (`pm.response.to.have.status(200)`) → Collection Runner. Run once with `Keep Variable Values` *off* = baseline; run again with vars on = fuzz pass. Watch for mid-run stalls (app choked on a payload — that request is interesting).

**Deep:** Intruder positions on every input surface of a single request; Wfuzz for speed. Method discovery = fuzz the HTTP verb itself (`§GET§` → PUT/POST/PATCH/DELETE/OPTIONS…) — a `400` where everything else is `405` means the method exists but the payload is wrong (then find the params it wants — Arjun).

## If you get banned mid-fuzz

Throttle (`wfuzz -t/-s`, Intruder Resource Pool), or pivot to evasion (`ch10`) — path mutations, header spoofing, IP rotation.
