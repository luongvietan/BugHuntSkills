# Ch08: MASVS-RESILIENCE — Anti-Tampering & Anti-Reversing

Source: MASTG tampering/RE chapter + platform resilience chapters, MASWE-0051..0065, resilience tests, best-practices MASTG-BEST-0029/0030/0041.

Framing: these controls raise attacker effort — they never stop a determined reverse engineer ("ultimately, the reverse engineer always wins"). Test them as a resilience assessment: you play the reverse engineer and try to defeat each control. Mostly an **MAS-L2** concern.

## Weakness types (MASWE)

| ID | Weakness | Core idea |
|---|---|---|
| MASWE-0051 | No root/jailbreak detection | Rooted/jailbroken devices run the app unchallenged |
| MASWE-0052 | No virtualization detection | App-in-app/virtual containers unhandled |
| MASWE-0053 | No emulator/virtual-device detection | Runs under emulators where instrumentation is trivial |
| MASWE-0054 | No device attestation | Backend can't verify a genuine device/OS state (Play Integrity/DeviceCheck/App Attest) |
| MASWE-0055 | No malware detection | Known-malicious co-installed apps ignored |
| MASWE-0056 | No app attestation / weak signing | Backend can't verify the binary is genuine; outdated signature scheme |
| MASWE-0057 | Resource integrity not verified | Modified assets/configs accepted |
| MASWE-0058 | Runtime code integrity not verified | Hooking/instrumentation undetected |
| MASWE-0059 | Insufficient code obfuscation | Security-relevant code trivially readable |
| MASWE-0060 | Resource obfuscation absent | Secrets/configs plainly visible |
| MASWE-0061 | Debug artifacts left in | Symbols, verbose logs, debug info in release |
| MASWE-0062 | No payload encryption at app level | Sensitive payloads rely on TLS only |
| MASWE-0063 | Debug mechanisms not disabled | `debuggable`, `get-task-allow`, WebView debugging in release |
| MASWE-0064 | No debugger detection | Debuggers attach freely |
| MASWE-0065 | No dynamic-analysis tool detection | Frida/Objection run undetected |

## Test procedures — the resilience assessment loop

For each claimed control: (1) find the check (static), (2) run against it (dynamic), (3) bypass it, (4) rate the effort required.

1. **Root/jailbreak detection**: Android checks — su binaries, Magisk artifacts, test-keys, RootBeer, SafetyNet/Play Integrity verdict; iOS — Cydia/Sileo presence, sandbox-write probes, fork/substrate detection. Bypass: Magisk DenyList/Zygisk hiding, objection `android root disable` / `ios jailbreak disable`, Frida scripts, Liberty Lite/Choicy. If your generic bypass wins, detection is weak.
2. **Emulator/virtual detection**: run on emulator → note blocks; virtualization apps (Parallel Space) and Corellium/cloud devices on iOS.
3. **Anti-debugging**: Android `Debug.isDebuggerConnected`, `ptrace` self-attach, TracerPid checks, `android:debuggable`; iOS `get-task-allow` entitlement (extract via `codesign -d --entitlements`), `PT_DENY_ATTACH`, sysctl. Try `lldb`/`jdb`/Frida spawn → attach succeeding unhindered = absent.
4. **Hook/integrity detection**: does the app detect Frida server (ports, maps scanning, named pipes, `gum-js-loop`)? Instrument anyway — no detection = MASWE-0058/0065 absent.
5. **Obfuscation quality**: decompile → are security-relevant classes (auth, crypto, license) readable? ProGuard/R8 strips names but keeps logic; real obfuscation (control-flow flattening, string encryption, packing via APKiD-detectable packers) raises effort. Evaluate *security-relevant* obfuscation, not cosmetic renames.
6. **Debug artifacts**: symbols in native binaries (`nm`, `rabin2`), debug logs (ch02 overlap), StrictMode logging, leftover test endpoints.
7. **Signature/attestation**: APK signature scheme version + key size (`apksigner verify`); iOS code signature format version; Play Integrity/App Attest/DeviceCheck actually enforced server-side (replay/tamper verdicts).
8. **Payload encryption**: app-level encryption of sensitive request bodies (banking-grade) — absence matters only where the threat model demands it.

## Bypass toolkit (you are the attacker)

Magisk (+Zygisk/DenyList/Shamiko), Xposed/LSPosed hooks, objection (`root`/`jailbreak`/`sslpinning`/`biometrics` disable commands), Frida codeshare scripts (universal unpinning, anti-anti-debug), JustTrustMe/SSLUnpinning/Android-SSL-TrustKiller, SSL Kill Switch 3 (iOS), RootCloak/RootBeer-evasion, patching the check out via apktool/smali or Hopper/Ghidra then re-signing, Frida Gadget injection for non-rooted/non-jailbroken coverage, Corellium for virtualized iOS.

## Interpretation

- Rate each control by **effort to defeat**: missing / stock-check (minutes) / custom multi-layer (days). Report controls absent *and* controls that fall to commodity bypasses.
- Resilience findings are **defense-in-depth gaps**, not vulnerabilities — absent root detection doesn't compromise users by itself; frame as maturity for L2 apps.
- Controls that only check at launch are weaker than runtime/continuous checks — note check timing and placement.
- Server-side attestation (Play Integrity/App Attest enforced in API) beats all client checks — verify the verdict is actually validated, not just collected.
