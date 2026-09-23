# Ch01: Testing Methodology, Platform Notes & Lab Setup

Source: MASTG general chapters (app taxonomy, security testing, tampering & RE), Android/iOS platform overviews, and the setup techniques (device shell, app extraction, proxy, pinning bypass).

## Testing types

- **Black-box** — no information beyond what's publicly discoverable; simulates an external attacker. Slow, least coverage.
- **White-box (full knowledge)** — source, docs, diagrams provided. Much faster and deeper; decompiled code helps but obfuscation makes decompilation costly, so always request source for a first-time test.
- **Gray-box** — credentials only; the industry default compromise between cost, speed, coverage.

**Static (SAST)** — review code without executing: manual review (grep for security-relevant APIs like `executeQuery`, `openURL`, `setAllowFileAccess`) + automated scanners for low-hanging fruit. Best at business logic, standards violations, design flaws (manual) and known-bad API patterns (automated).

**Dynamic (DAST)** — exercise the running app on device/emulator; analyze real behavior, filesystem changes, traffic, IPC. Catches what static review can only hypothesize.

The MASTG test taxonomy pairs these: **"References to X APIs"** tests = static; **"Runtime use of X"** tests = dynamic. Run both.

## Pentest phases (MASTG)

1. **Preparation** — scope, security level (MAS-L1 baseline for all apps; MAS-L2 for apps handling sensitive data or under regulation), legal protection for the tester. Attacking systems without written authorization is illegal.
2. **Intelligence gathering** — environmental (org goals, industry, processes → business-logic leads) and architectural (app data flows, OS versions, TLS/pinning posture, remote services).
3. **Mapping** — entry points, features, data; rank potential vulns by damage; write test cases.
4. **Exploitation** — confirm findings are real, not scanner noise.
5. **Reporting** — exploitation detail, vuln class, risk, accessed data.

## Setup essentials

- **Two build variants**: a release build (verifies controls work) and a debug build with controls disabled (lets you cover the rest). Request both; testing each doubles effective coverage.
- **Devices**: rooted Android (Magisk) or jailbroken iOS device for full filesystem/instrumentation access; emulators where acceptable. Company policy may restrict rooted devices on client premises — coordinate in preparation. Non-rooted dynamic analysis is still possible via repackaged apps with injected instrumentation (Frida Gadget).
- **Sensitive-data definition** (agree before testing): credentials/PINs, PII (SSN, card, bank, health), person-identifying device IDs, reputation/financial-impact data, legally protected data, and technical secrets the app generates (keys, tokens).
- **Data states**: at rest (files/stores), in use (memory), in transit (network and IPC). Scrutiny scales with importance and exposure likelihood.

## False-positive discipline

Automated scanners apply web assumptions to mobile. Classic misfires:
- **CSRF** — needs a browser auto-attaching cookies; mobile apps don't share the browser cookie store. Almost never a real mobile bug.
- **Reflected XSS** — links open in the default browser, not the app's WebView. Stored XSS matters only when the app renders in a WebView with a JS bridge.
- **Weak crypto in non-security context** — an insecure RNG for a game shuffle is fine. Name the security-relevant context before reporting.

## Platform notes — Android

- **Package**: APK/AAB containing `AndroidManifest.xml` (package name, permissions, component declarations, `networkSecurityConfig`), `classes.dex`, `resources`/`res`, `assets`, `lib/` native code, `META-INF` signature. `apktool d` unpacks; `jadx` decompiles dex→Java; re-sign modified APKs with `apksigner`/`uber-apk-signer`.
- **Sandbox**: each app a Linux UID; private dir `/data/data/<pkg>` (internal storage). External storage (`/sdcard`) is shared world-readable — never for secrets.
- **IPC entry points** (exported = reachable by other apps): **Activities** (UI screens), **Services** (background), **Broadcast Receivers** (pub/sub), **Content Providers** (`content://` CRUD over structured data). Export via `android:exported`, intent-filters, or legacy defaults; protect with permissions/signature-level checks.
- **Intents**: explicit (named component) vs implicit (resolved by intent-filter — interceptable). PendingIntents and deep links (`myapp://`, App Links) ride the same channel.
- **Keystore**: hardware-backed (TEE/StrongBox) key storage via Android Keystore; keys non-exportable, can require biometric unlock and invalidate on new enrollment.
- **Access**: `adb shell`, `run-as` on debuggable builds, `adb backup` (deprecated path — check `allowBackup`), `content` CLI for providers, `am`/`pm` for components/packages.

## Platform notes — iOS

- **Package**: IPA zip containing the Mach-O binary, `Info.plist`, `embedded.mobileprovision`, entitlements, frameworks, resources. Encrypted App Store binaries must be dumped first (frida-ios-dump on a jailbroken device, or pull the decrypted IPA from device backup paths).
- **Sandbox**: app container `Data/` (Documents, Library, tmp) + `Bundle/`; **Data Protection classes** encrypt files per class key tied to passcode — `Complete` / `CompleteUnlessOpen` / `CompleteUntilFirstAuthentication` / `None`.
- **Keychain**: secure enclave–backed secret store, accessibility flags (`kSecAttrAccessible*`), Access Control flags for biometric-gated items. Dump with Keychain-Dumper/objection on jailbroken devices.
- **IPC**: URL schemes, universal links (domain-verified), app extensions, App Groups shared containers, pasteboard, XPC services — much narrower than Android IPC, so misuse concentrates on URL-scheme/universal-link validation and pasteboard/App Group leakage.
- **ATS**: App Transport Security enforces TLS by default; `Info.plist` `NSAppTransportSecurity` exceptions (per-domain `NSExceptionAllowsInsecureHTTPLoads`, `NSAllowsArbitraryLoads`) are audit targets.
- **Access**: SSH on jailbroken devices; `iproxy`/`usbmuxd` port forwarding; `libimobiledevice` suite for install/backup/syslog; `simctl` for simulator.

## First-touch checklist

Proxy configured + CA installed → app exercised end-to-end (traffic map) → pinning bypassed if needed → package extracted (apktool / IPA dump) → manifest/Info.plist reviewed (components, ATS/NSC, permissions, schemes) → data dirs + logs + backup checked for sensitive data → MASVS category chapters executed per scope.
