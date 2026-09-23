# Cheatsheet — OWASP MASTG

Quick-lookup card. Details in `chapters/`; mindset in `SKILL.md`; terms in `glossary.md`.

## Session kickoff

1. Confirm authorization + scope; define sensitive data; request release + debug builds.
2. Root/jailbreak device or emulator; install app; proxy + CA cert; verify traffic.
3. Bypass pinning if needed (objection → Frida script → patch).
4. Extract package (`apktool d` / IPA dump); read manifest/Info.plist first.
5. Baseline data-dir snapshot → exercise app fully → diff files, watch logs, map endpoints.

## Category → first probes

| MASVS | Probe | Fail condition |
|---|---|---|
| STORAGE | diff app dirs before/after exercising; `adb logcat`; `adb backup`/iTunes backup | plaintext secrets in files/logs/backup/external storage |
| CRYPTO | grep `Cipher.getInstance`, `MD5`, `Random`, key literals; hook `doFinal`/`CCCrypt` | broken algo/ECB/fixed IV/hardcoded key/non-CSPRNG for secrets |
| AUTH | hook biometric callback → does a boolean flip unlock? replay tokens post-logout | UI-bound biometrics; tokens valid after expiry/logout |
| NETWORK | proxy + rogue cert; sniff for HTTP; read NSC/ATS | cleartext sensitive data; rogue CA accepted; accept-all TrustManager |
| PLATFORM | `am start` exported comps; fire crafted deep links; WebView bridge probe | unprotected exported sensitive function; unvalidated link/bridge/file access |
| CODE | `apksigner verify`; checksec on `.so`; provider `content query` injection | missing PIE/canary; reachable injection; vuln dep code path hit |
| RESILIENCE | run on root+Frida; attach debugger; decompile | no root/hook/debug detection where expected; debuggable release |

## Commands

```bash
adb devices -l && adb install app.apk
adb shell pm path pkg.name && adb pull <path>          # grab APK
apktool d app.apk && jadx -d out app.apk                # unpack + decompile
adb shell am start -n pkg/.ActivityName                 # invoke exported activity
adb shell am start -a android.intent.action.VIEW -d "scheme://x"
adb shell content query --uri content://pkg.provider/table
adb logcat | grep -i pkg ; pidcat pkg.name              # log watching
adb backup -f app.ab pkg.name                           # backup test
apksigner verify --verbose app.apk                      # signature scheme
frida -U -f pkg.name -l hook.js ; objection -g pkg.name explore
objection → android sslpinning disable / ios sslpinning disable
objection → ios jailbreak disable / android root disable
objection → env | memory dump | ios keychain dump | ios nsuserdefaults get
frida-ios-dump → decrypt IPA ; iproxy 2222 22 → ssh device
idevicesyslog | idevicebackup2 backup --full ./bak      # iOS logs/backup
checksec --file=lib.so ; rabin2 -I binary               # binary protections
testssl.sh https://api.target ; nscurl --ats-diagnostics https://api.target
MobSF: upload APK/IPA → static+dynamic report           # baseline scan
cdxgen -o sbom.json → dependency-check                  # dep vulns
```

## Bypass quick ref

- **Pinning**: objection toggle → Frida codeshare unpinning → JustTrustMe/SSL Kill Switch → patch check → reFlutter for Flutter.
- **Root/jailbreak detect**: DenyList/Zygisk, objection disable cmds, patch the check, Liberty Lite/Choicy.
- **No root/jailbreak**: repackage with Frida Gadget or injected lib, re-sign, reinstall.
- **Debugger blocked**: `am set-debug-app -w` / `am start -D`, ptrace timing, attach post-launch, patch anti-debug.
- **Obfuscated code**: go dynamic — trace calls, dump memory, hook crypto outputs.

## Interpretation quick ref

- Exported + sensitive + no permission = finding; exported launcher = noise.
- Cleartext or rogue-cert acceptance = high; missing pinning = L2-only gap.
- UI-bound biometric bypass = real bug; crypto-bound survives the hook.
- Vuln dep needs a reachable path; debug artifacts need a release build.
- Resilience gaps are maturity findings (effort rating), not user-facing vulns.

## When stuck

- Switch category — storage findings chain into auth/network leads.
- Old app versions, alternate architectures (`lib/armeabi`), split APKs.
- Read the decompiled code harder — the undocumented endpoint/param is there.
- Pair static "references" hits with runtime confirmation you haven't done yet.
