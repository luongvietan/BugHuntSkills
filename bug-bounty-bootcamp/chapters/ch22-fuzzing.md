# Ch25: Automatic Vulnerability Discovery Using Fuzzers

Source: Chapter 25. Fuzzing = send masses of invalid/unexpected input, watch for anomalies. Li's framing: manual first (learn the target, find logic bugs), then automate the grind — fuzzers are *metal detectors*, not metal.

## The 4-step process

1. **Find data injection points** — every place input enters: URL params, POST bodies, headers, cookies, paths. Classify each by likely vuln (numeric ID → IDOR; search → XSS; URL fetch → SSRF).
2. **Pick the payload list** — matched to the vuln class:
   - **SecLists** (Daniel Miessler) — directories, params, vuln payloads
   - **FuzzDB**, **Big List of Naughty Strings** — edge cases
   - Payloads per class: XSS list, SQLi list, ID ranges, traversal strings
   - Also *random* garbage:超长 strings, odd encodings, newlines/nulls/special chars — finds novel bug classes via crashes/anomalies.
3. **Fuzz systematically** — tools:
   - **Burp Intruder**: right-click request → Send to Intruder → mark positions (`Add`) → payload sets (list/numbers/random/brute-forcer) → attack types (sniper single-point, cluster bomb multi-point). Free version is throttled.
   - **Wfuzz**: CLI, unthrottled — `FUZZ`/`FUZ2Z` markers map wordlists to positions.
   - **ZAP fuzzer**: free alternative.
4. **Monitor for anomalies** — sort results by: status code (200 vs 404 = path found), **response length** (different = content changed → possible injection), **time** (delay payloads → blind injection), error strings. One anomalous row = lead for manual follow-up.

## Wfuzz cookbook

```
# Path enumeration — hide 404s, follow redirects
wfuzz -w wordlist.txt -f out.txt --hc 404 --follow http://target/FUZZ

# IDOR scan — iterate IDs, diff response length
wfuzz -w ids.txt http://target/view?user_id=FUZZ

# Basic-auth brute force — FUZZ:user FUZ2Z:pass
wfuzz -w users.txt -w pass.txt --basic FUZZ:FUZ2Z http://target/admin

# Custom header fuzzing (auth bypass, UA tricks)
wfuzz -w agents.txt -H "User-Agent: FUZZ" http://target/admin

# Open redirect — watch for Location anomalies
wfuzz -w redirs.txt -v --follow "http://target/?redirect=FUZZ"

# XSS — filter for reflected payload
wfuzz -w xss.txt --filter "content~FUZZ" "http://target/?q=FUZZ"

# SQLi via POST — watch time/length anomalies
wfuzz -w sqli.txt -d "user_id=FUZZ" http://target/get_user
```

## Fuzzing vs. static analysis — and limits

Fuzzing needs no source and sees live behavior; static review (ch19) needs code but sees all paths. Fuzzing **misses**: business-logic errors, multi-step attacks, anything needing valid state. Fuzzing **risks**: rate limits/bans, accidental DoS (throttle + get permission), noisy logs. Rule: understand every tool's internals (read Sublist3r/Wfuzz source — they're small Python) before trusting its output.

## Checklist

Injection points classified → payload lists matched per class → fuzz tool chosen → throttle/permission set → results sorted by code/length/time → anomalies manually verified → findings reported with reproducible single-request PoC.
