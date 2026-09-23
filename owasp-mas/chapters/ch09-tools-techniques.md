# Ch09: Tools & Techniques — The MASTG Toolbox

Source: MASTG tools catalog + techniques pages (device access, analysis, MITM, instrumentation, patching).

> **Tool currency (2026 refresh):** tool names and flags drift — check each
> tool's current release before scripting (Frida/Objection versions pair
> with OS versions; MobSF, apktool, jadx, idevicebackup2 flags change).
> MASTG upstream lists `MASTG-TOOL-*` entries per tool — prefer those over
> memorized invocations.

## Core instruments

**Frida** — runtime instrumentation engine. `frida-server` on rooted/jailbroken device; `frida -U -f pkg` spawn + script hooks. Core uses: hook Java/ObjC methods (`Java.use`, `ObjC.classes`), trace crypto/network APIs, dump runtime values, call app functions. **Frida Gadget** (injected lib) enables instrumentation on **non-rooted/non-jailbroken** devices via repackaging. Frida CodeShare = community scripts (universal SSL unpinning, root-detection bypass, crypto monitors). r2frida ties it to radare2; Fridump dumps process memory.

**Objection** — Frida-powered CLI automation, zero scripting: `objection -g pkg explore` → `android sslpinning disable`, `ios sslpinning disable`, `android root disable`, `ios jailbreak disable`, `ios ui biometrics_bypass`, `env`, `android hooking list activities/services/receivers`, `memory dump`, `ios nsuserdefaults get`, `ios keychain dump`, `android intent launch_activity`. First tool for routine bypass + recon at runtime.

**MobSF** — automated static+dynamic analyzer for APK/IPA. Import the package → manifest analysis, code findings, hardcoded-secrets scan, tracker/permission audit, binary protections, SBOM-ish dependency view. Great baseline; verify findings manually (false positives).

## Android toolchain

- **adb** — `devices`, `shell`, `install`, `pull`/`push`, `logcat`, `run-as`, `am`/`pm`/`content`/`dumpsys`, `backup`.
- **apktool** — decode/rebuild APK (smali, manifest, resources); `aapt2`/`apksigner`/`uber-apk-signer` inspect + re-sign.
- **jadx / Bytecode Viewer / APKLab** — dex→Java decompilation + cross-references; **apkx** wrapper decompile; **APKiD** identifies packers/obfuscators/Compilers; **Apkleaks** greps URIs/secrets.
- **Magisk** — systemless root + Zygisk hiding; **Xposed/LSPosed** — module hooks (JustTrustMe, RootCloak).
- **drozer** — component/IPC attack surface enumeration + exploitation (`run app.provider.query`, `app.activity.start`).
- **House**, **jnitrace**, **RMS (Runtime Mobile Security)** — Frida-based tracing UIs; **lldb/jdb** — debugging; **Ghidra/radare2/iaito** — native RE; **angr/Angr** — symbolic execution; **Termux/Busybox/scrcpy/pidcat/FlowDroid** — on-device shell, utilities, screen share, log filtering, taint analysis.

## iOS toolchain

- **Jailbreak** — checkra1n/palera1n family; package managers Cydia/Sileo; tweaks via ElleKit/substitute; Choicy per-app tweak control.
- **libimobiledevice suite** — `ideviceinstaller`, `idevicebackup2`, `idevicesyslog`, `iproxy` (usbmuxd port forward), `ios-deploy`; **Filza** file manager; **iOSbackup** backup extraction.
- **App handling** — frida-ios-dump (decrypt App Store binaries), ipainstaller/Sideloadly/ios-app-signer/ldid/codesign for install + re-sign, `ipsw` for firmware.
- **Analysis** — `otool`/`objdump`/`nm`/`class-dump*`/`dsdump`/`swift-demangle` (Mach-O + ObjC/Swift metadata), **Keychain-Dumper**, **BinaryCookieReader**, **Plutil/PlistBuddy/plistlib**, `security` CLI, Grapefruit (Frida UI), Cycript, MachoOView, Hopper/Ghidra/lldb, **SSL Kill Switch 3** (TLS validation disable), **IOSSecuritySuite** (defensive checks to study).
- **xpcspy** — XPC message tracing; `simctl` + Xcode instruments for simulator workflows.

## Generic techniques map

- **Get the app**: Play/ADB pull (`adb shell pm path` + `adb pull`), gplaycli/apkeep (Play download); iOS: frida-ios-dump or device backup extraction.
- **Static pipeline**: unpack (apktool / unzip IPA) → manifest/Info.plist/entitlements → decompile (jadx / Ghidra / class-dump) → strings + cross-references → API-usage grep → secrets scan (gitleaks/Apkleaks) → SCA/SBOM (cdxgen, dependency-check).
- **Dynamic pipeline**: install → baseline data-dir snapshot → exercise app → monitor logcat/syslog → watch filesystem deltas → trace with Frida (method trace, execution trace, native trace, JNI trace) → inspect open files/connections/loaded libs → memory dump.
- **MITM ladder** (escalate until traffic is visible): system proxy → CA installed to system store → network-layer capture (tcpdump/Wireshark, ARP spoof/rogue AP via bettercap/hostapd) → hook TLS functions (`SSL_read`/`SSL_write`) → hook app-layer network APIs → repackage with instrumentation.
- **Pinning bypass ladder**: objection toggle → Frida codeshare universal bypass → platform-specific tools (JustTrustMe, SSL Kill Switch) → patch the pinning code → Flutter: reFlutter/disable-flutter-tls-verification.
- **Patching/repackaging**: edit smali/decompiled code or binary (Ghidra/radare2 patch), rebuild (`apktool b`), re-sign (apksigner/uber-apk-signer; codesign+ldid+ios-app-signer on iOS), reinstall. Wait-for-debugger (`am set-debug-app -w pkg` or `am start -D`) for early instrumentation.
- **Framework notes**: Flutter (no system proxy, own pinning, Dart AOT — blutter/reFlutter), React Native (Hermes bytecode — hermes-dec; JS bundle patching), Xamarin (own TLS stack), hybrid WebView apps (ch06 surface).

## Choosing the approach

| Situation | Path |
|---|---|
| Rooted/jailbroken device available | frida-server/objection directly |
| Stock device | Repackage + Frida Gadget / injected lib |
| Heavy obfuscation/packing | Dynamic-first: trace runtime, dump memory, patch checks |
| Quick baseline | MobSF scan + manifest review + proxy traffic |
| Backend API focus | Proxy + pinning bypass → then web/API methodology |
