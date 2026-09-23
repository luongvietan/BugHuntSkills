# Ch13: Evasive Techniques & Rate Limit Testing

Source: Chapter 13. Controls block based on *attribution* — IP, token, headers, request metadata. Change what they attribute → evade.

> **Gating (2026 refresh):** this chapter is *not* quickstart material. Evasion
> probes (IP rotation, header spoofing, encoding tricks) and any resource-
> exhaustion test require **explicit permission in the policy** — many programs
> ban automated evasion or DoS-class testing outright. Rate-limit *verification*
> is different from evasion: a small number of extra requests past the limit on
> your own account demonstrates the mechanism; rotating infrastructure to keep
> going is an escalation that needs permission. Omit these from quickstarts and
> first passes.

## Detecting controls before they detect you

Use the API as intended first; controls hide in responses: `X-CDN: Imperva|Zenedge|fastly|akamai|Incapsula`, `X-Kong-Proxy-Latency`, `Server: Zenedge/Kestrel`, `X-Original-URI`. A 302 to a CDN = traffic proxied through a WAF. Active detection: `nmap --script http-waf-detect`, Wafw00f, W3af, Bypass WAF.

WAF triggers: too many 404s, high request rate, known-attack signatures (SQLi/XSS), abnormal authZ probes — each with a threshold (e.g., 3 strikes → block).

**Burner accounts**: register several disposable accounts *before* attacking (unique emails/names, different IP if warranted). Probe thresholds with burners, learn the triggers, keep main accounts clean.

## Evasion techniques

- **String terminators** — `%00`, `0x00`, `//`, `;`, `%`, `!`, `?`, `[]`, `%5B%5D`, `%09`, `%0a`, `%0b`, `%0c`, `%0e`. Null byte ends server-side string processing → filter never sees the rest: `<s%00cript>alert(1)</s%00cript>`.
- **Case switching** — `<sCriPt>`, `SeLeCT`, `sELecT @@vErSion`. Beats naive signature rules.
- **Encoding** — URL-encode blocked chars (`< > ( ) [ ] { } ; ' / \ |`), partial or full payload, HTML/base64 variants; **double-encode** when a front control decodes once and backend decodes again.
- **Automate with Intruder Payload Processing** — ordered rules (top-down): e.g., ① Encode URL-All-Characters ② Add Prefix `%00` ③ Add Suffix `%00`. Verify in the Payload column.
- **Wfuzz encoders** — `wfuzz -e encoders` lists all; `-z file,list.txt,base64` per-payload encode; `-z list,a-b-c,base64-md5-none` = one request per encoder (3×3=9); `-z list,aaa-bbb,base64@random_upper` = chained (random_upper→base64, one request each).

## Rate-limit testing

**Existence:** docs/marketing, `x-rate-limit:` / `x-rate-limit-remaining:` headers, or push until 429/`Retry-After`/ban.

**Lax limits are findings:** 15,000 req/min doesn't stop a 150,000-word brute force — just throttle: Wfuzz `-s 6` (≈10/min) or Intruder Resource Pool delay ms (100ms≈10/s). Stay under the limit and the attack itself proves the limit is weak.

**Bypasses once limited:**

- **Path bypass** — mutate the URL: `POST /api/myprofile%00`, `%20`, `/api/myProfile`, `/api/MyProfile`, `/api/my-profile`, or junk param `/api/myprofile?test=§N§` (pitchfork: increment N alongside the real payload — rate counter may key on exact path+query).
- **Origin header spoofing** — `X-Forwarded-For`, `X-Forwarded-Host`, `X-Host`, `X-Originating-IP`, `X-Remote-IP`, `X-Client-IP`, `X-Remote-Addr` with 127.0.0.1/private/target-range IPs, one at a time or all (431 = too many headers, trim). `User-Agent` rotation via SecLists UserAgents.fuzz.txt. Success = `x-rate-limit-remaining` resets or requests succeed post-block.
- **Token/identity rotation** — 20k tokens from a Sequencer live capture = 20k identities if old tokens stay valid.
- **IP Rotate extension** — proxies Burp traffic through AWS API Gateway; every request exits a different AWS IP. Real rotation, not spoofing. Needs Jython + boto3 + IAM user (APIGatewayAdministrator + InvokeFullAccess). Test against ipchicken.com.
- **Client rotation** — different account/token hits a different limit bucket.
