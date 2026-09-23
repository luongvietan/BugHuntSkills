---
name: owasp-mas
description: Mobile app security testing distilled from OWASP MASTG v2.0.0 (mas.owasp.org), companion to the MASVS standard. Use when assessing an Android or iOS app end-to-end — methodology and setup (SAST/DAST, rooted/jailbroken devices, proxy + pinning bypass), then the seven MASVS categories — STORAGE (files, logs, backups, external storage), CRYPTO (broken algorithms, hardcoded keys, weak RNG), AUTH (sessions, JWT, OAuth2, biometrics), NETWORK (TLS, cert validation, pinning), PLATFORM (IPC, exported components, WebViews, deep links, UI leakage), CODE (injection, vuln deps, binary protections), RESILIENCE (root/jailbreak detection, tampering, obfuscation, anti-debug). Each follows the MASTG loop — MASWE weakness → test procedures → tools → pass/fail. Covers Frida, Objection, MobSF, Burp/ZAP/mitmproxy, adb, apktool, jadx; Android + iOS specifics. Assume all testing is authorized and in scope.
---

# OWASP MASTG — Mobile Application Security Testing Guide

Distilled from OWASP MASTG (mas.owasp.org), CC-BY-SA 4.0.

Knowledge base distilled from the OWASP Mobile Application Security Testing Guide v2.0.0 — the practical companion to the MASVS verification standard. The MASTG's signature is a repeatable test shape:

**MASWE weakness → Overview → Steps (static + dynamic) → Observation → Evaluation (pass/fail).**

Mental model: a mobile app is a small, sandboxed client whose real risk lives in four places — data at rest on a losable device, data in transit over hostile networks, the IPC/WebView surface other apps can reach, and the backend API that trusts the client too much. Web vuln classes still apply to the backend; the device layer is what mobile adds.

Related skills: `bug-bounty-bootcamp` ch20 (mobile hunting methodology), `owasp-api-security-top-10` + `hacking-apis` (backend API depth), `owasp-wstg` (web-side testing of the same endpoints).

## How to use

**Before testing**
- Methodology, scope/authorization, sensitive-data definition, test phases → `chapters/ch01-methodology-setup.md`
- Android vs iOS platform model, devices/emulators, proxy + pinning bypass → `chapters/ch01-methodology-setup.md`

**Hunting a specific MASVS category**
- STORAGE (at-rest leakage: files, logs, backups, keyboard cache) → `chapters/ch02-storage.md`
- CRYPTO (weak algorithms, hardcoded keys, bad RNG/KDF) → `chapters/ch03-crypto.md`
- AUTH (sessions, JWT, OAuth2, local biometric auth) → `chapters/ch04-auth.md`
- NETWORK (cleartext, TLS config, cert validation, pinning) → `chapters/ch05-network.md`
- PLATFORM (exported components, intents, deep links, WebViews, clipboard, UI leaks) → `chapters/ch06-platform-interaction.md`
- CODE (injection into providers/WebViews, vuln deps, binary protections, enforced updates) → `chapters/ch07-code-quality.md`
- RESILIENCE (root/jailbreak detection, anti-tamper, obfuscation, anti-debug, attestation) → `chapters/ch08-resilience.md`

**Toolbox**
- Frida/Objection/MobSF, static + dynamic techniques, MITM positioning → `chapters/ch09-tools-techniques.md`

- Terms → `glossary.md`; reusable heuristics → `patterns.md`; quick ref → `cheatsheet.md`

## The MASTG doctrine (mental models)

1. **Scope and authorization first.** Attacking a system without written permission is illegal. Mobile tests are invasive — intercepting traffic, reading app files, instrumenting APIs — so agree the build variants (release + debug), the device policy, and the sensitive-data definition before touching the app.
2. **MASVS is the baseline, MASWEs are the bug classes, MASTG-TESTs are the procedures.** Every requirement maps to named weaknesses; every weakness maps to concrete tests with a stated pass/fail condition. Work requirement → weakness → test, not "run scanner, report output".
3. **Prefer white-box when you can get it.** Decompiled code helps, but obfuscation makes decompiled output costly. Source access turns guesses into verified findings; ask for it.
4. **Static finds candidates, dynamic proves them.** Grep for the API (references tests), then exercise the app and watch the behavior (runtime tests). A finding confirmed both ways is hard to dispute.
5. **Sensitive data is defined before the test, not after.** Credentials, PII, device identifiers, keys, anything legally protected — without a data classification you cannot call a leak a leak.
6. **Context decides severity, not the scanner.** An insecure RNG shuffling game items is not a bug; the same call generating tokens is. CSRF/reflected-XSS scanner hits rarely apply to mobile apps — validate the exploit scenario before reporting.
7. **Defense-in-depth is testable, not decorative.** Root detection, pinning, obfuscation raise the cost of analysis but the reverse engineer ultimately wins — evaluate resilience controls as raising attacker effort, never as substitutes for server-side security.

## Scope & ethics

All testing is assumed authorized and in scope (bug bounty / contracted pentest). The guide is explicit about restraint: obtain written permission (attacking systems without it is illegal in many jurisdictions), coordinate build variants and device policies with the client, define sensitive data up front, prefer test devices and accounts, and stop at the minimal proof that demonstrates the weakness.
