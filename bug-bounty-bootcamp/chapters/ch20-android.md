# Ch23: Hacking Android Apps

> **Era note:** the IPC/storage/webview *classes* are durable; the systematic current procedure (MASWE → test → tools) lives in `owasp-mas` (MASTG v2.0.0) — use this chapter for concept grounding, that skill for the actual assessment.

Source: Chapter 23. Mobile hacking ≈ web hacking + device/tooling layer. Fewer hunters per program → less duplication, more bugs. OWASP MSTG is the deep reference.

## Setup

- **Proxy**: Burp listener on `All interfaces` + port; Android Wi-Fi → Modify Network → proxy = laptop IP (`hostname -i` / `ipconfig getifaddr en0`) : port. Install Burp CA on device (Settings → Security → Install from storage → "VPN and apps"). Don't do this on public Wi-Fi.
- **Dedicated device or emulator** (Android Studio emulator) — testing can void warranty/damage the device.
- **Cert pinning bypass**: if proxying shows nothing, app pins certs. Try **Objection** `android sslpinning disable`, or **Frida** + Universal Android SSL Pinning Bypass script (codeshare). Custom pinning → patch the app or its validation code manually.

## APK anatomy (where to look)

`apktool d app.apk` unpacks:

- **AndroidManifest.xml** — first read: package, permissions (SMS/contacts → sensitive capability), and exported components: **Activities** (UI screens — exported = launchable by other apps), **Services** (background work), **BroadcastReceivers** (respond to system/app broadcasts — can be triggered maliciously), **ContentProviders** (share data — exported = other apps read your data).
- **classes.dex** — compiled code; decompile (JADX/jd-gui) → source review per ch19.
- **res/values/strings.xml** — hardcoded URLs, keys, secrets.
- **lib/** — native code per architecture; **assets/** — bundled files/configs; **META-INF/** — signature.
- `apktool b dir -o mod.apk` repackages after edits; re-sign + `adb install` to test modified behavior.

## Toolkit

ADB (`adb devices -l`, `install`, `pull`/`push`, shell), Android Studio (emulator + code edit), Apktool, JADX, Frida (runtime instrumentation — hook functions, trace calls), Objection (Frida automation), MobSF (full static+dynamic auto-analysis).

## Hunting — mobile-specific angles

1. **Exported components**: unprotected exported activity/receiver/provider = other apps (or attacker links) invoke privileged functions or read provider data. Test with `adb shell am start -n pkg/.Activity` and intent fuzzing.
2. **Session handling**: mobile apps use long-lived/non-expiring tokens, reuse tokens — chain: token theft → persistent access even after password change.
3. **Hardcoded secrets**: API keys in strings.xml/dex; custom crypto, weak algorithms, hardcoded encryption keys.
4. **Mobile-vs-web parity gaps**: mobile endpoints trust the app → IDOR/auth flaws absent on web; older API versions called by app.
5. **Local storage**: sensitive data in SharedPreferences/SQLite/logs on rooted devices.
6. **Deep links & WebViews**: `myapp://` intents with URL params → XSS/redirect inside app; WebView `addJavascriptInterface` bridges = RCE-adjacent.

## Checklist

Proxy traffic mapped → pinning bypassed → manifest reviewed for exported/leaky components → strings/dex reviewed for secrets → web-vuln suite (ch4-18) run against API endpoints → token/session model attacked → dynamic validation on emulator.
