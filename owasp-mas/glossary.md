# Glossary — OWASP MAS / MASTG

Terms as the OWASP Mobile Application Security project uses them. Backend/web terms live in `owasp-wstg`/`owasp-api-security-top-10` glossaries.

## Project vocabulary

- **MASVS** — Mobile Application Security Verification Standard: the requirements catalog, split into 8 control groups (STORAGE, CRYPTO, AUTH, NETWORK, PLATFORM, CODE, RESILIENCE, PRIVACY).
- **MASTG** — this testing guide: procedures proving or disproving each MASVS requirement; organized as techniques (MASTG-TECH), tests (MASTG-TEST), tools (MASTG-TOOL), knowledge (MASTG-KNOW), best practices (MASTG-BEST).
- **MASWE** — Mobile Application Security Weakness Enumeration: named weakness types bridging MASVS requirements and concrete tests (e.g., MASWE-0026 Network Traffic Not Encrypted).
- **MAS-L1 / MAS-L2** — assurance profiles: L1 = baseline for all apps; L2 = apps handling sensitive data or in regulated industries (adds resilience + pinning requirements).
- **"References to X" vs "Runtime use of X" tests** — the v2 test split: static grep of decompiled/config for the API vs dynamic confirmation the code path executes.
- **Sensitive data** — the agreed pre-test list: credentials, PII, device identifiers, legally protected data, app-generated secrets. Undefined ⇒ leakage can't be called a finding.
- **Resilience assessment** — the tester-as-reverse-engineer exercise rating how hard anti-tamper controls are to defeat.

## Platform vocabulary

- **APK / AAB / IPA** — Android install packages / iOS archive; contain manifest/plist, code, resources, signature.
- **AndroidManifest.xml** — declares package, permissions, components + exported flags, intent filters, `networkSecurityConfig`, `allowBackup`, `debuggable`.
- **Exported component** — activity/service/receiver/provider invocable by other apps; an IPC entry point needing permission checks.
- **Intent** — Android IPC message; explicit (named target) or implicit (resolved via intent-filter — interceptable). Carries extras (untrusted input).
- **Content provider** — `content://` CRUD interface over app data; export + missing protection = data theft/SQLi.
- **Deep link / App Link / custom scheme** — URL-driven entry points (`myapp://`, verified https links); need param + source validation.
- **Universal link** — iOS verified-domain links backed by `apple-app-site-association`.
- **Keystore / Keychain / Secure Enclave** — hardware-backed secret storage; keys non-exportable, biometric-gateable.
- **Data Protection classes** — iOS per-file encryption tiers tied to passcode state (`Complete`, `CompleteUnlessOpen`, `UntilFirstAuthentication`, `None`).
- **ATS / NSC** — iOS App Transport Security / Android Network Security Config: declarative TLS + cleartext + trust-anchor + pinning policy.
- **App Group / sandbox container** — iOS shared containers across app+extensions; Android sandbox = per-UID `/data/data/<pkg>`.
- **BiometricPrompt / LAContext** — local biometric auth APIs; crypto-bound (unlocks a key) vs UI-bound (returns boolean — hookable).
- **WebView bridge** — `addJavascriptInterface` / `WKScriptMessageHandler`: native APIs exposed to web content.
- **Pasteboard / clipboard** — shared copy buffer readable by any app unless scoped/expired.
- **Play Integrity / App Attest / DeviceCheck** — server-verifiable device+app attestation.

## Technique vocabulary

- **Root / jailbreak** — privilege escape enabling filesystem + instrumentation access; Magisk (Android), checkra1n/palera1n (iOS).
- **Certificate/identity pinning** — app trusts a pinned SPKI/cert set instead of any CA-valid chain; bypass required to proxy.
- **Frida / Gadget** — runtime instrumentation engine; Gadget = injected library enabling hooks on unmodified stock devices.
- **Objection** — Frida automation CLI (pinning/root/jailbreak/biometric toggles, exploration).
- **MITM ladder** — proxy → CA trust → packet capture/ARP/rogue-AP → TLS-function hooking → app-layer hooking → repackage.
- **SAST / DAST** — static (code/config review) vs dynamic (runtime observation) analysis.
- **SBOM / SCA** — software bill of materials / dependency vuln scanning (cdxgen, dependency-check).
- **Smali / dex** — Android bytecode assembly/compiled form; apktool↔smali is the patch loop.
- **Mach-O / entitlements** — iOS binary format and its granted capabilities (incl. `get-task-allow` debug flag).
- **Tampering vs reversing** — modifying the app/process to change behavior vs analyzing it to understand behavior.
- **MAS Crackmes** — the project's practice targets for reversing skills.
