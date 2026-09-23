# Ch07: MASVS-CODE — Code Quality & Build Settings

Source: MASTG code-quality chapter (injection, XSS, memory corruption, binary protections), MASWE-0041..0050, code tests, best-practices MASTG-BEST-0006/0007/0010/0021/0022/0039.

## Weakness types (MASWE)

| ID | Weakness | Core idea |
|---|---|---|
| MASWE-0041 | Recent platform version not ensured | App runs on OS versions lacking current security features |
| MASWE-0042 | Latest platform version not targeted | `targetSdkVersion`/SDK behind → legacy behaviors apply |
| MASWE-0043 | Enforced updating not implemented | No forced-update path → vulnerable versions stay in the wild |
| MASWE-0044 | Dependencies with known vulnerabilities | Vulnerable third-party libs (SCA/SBOM finding) |
| MASWE-0045 | Compiler security features unused | Missing PIE, stack canaries, ARC, etc. in native code |
| MASWE-0046 | Deprecated APIs in use | e.g., UIWebView, old crypto APIs |
| MASWE-0047 | Non-standard APIs for security-critical code | Custom/undocumented APIs where platform provides vetted ones |
| MASWE-0048 | Malicious code included | Trojanized deps/SDK |
| MASWE-0049 | Unsafe dynamic code loading | `DexClassLoader`/`dlopen` of remote or writable code |
| MASWE-0050 | Unsafe handling of untrusted data | Injection into SQL/XML/serialization from external input |

## Injection & untrusted data (MASWE-0050)

Mobile's injection surface is narrower than web but real:
- **SQL injection** — Content Provider `selection`/`projection` args built from input; raw `SQLiteDatabase.execSQL`/Room `@RawQuery`; test provider URIs for traversal (`content://provider/../../`) and injection characters.
- **XML injection/XXE** — XML parsers on device handling untrusted XML (rare; check `XmlPullParser`, `DocumentBuilderFactory` without `FEATURE_SECURE_PROCESSING`, iOS `NSXMLParser`).
- **Unsafe deserialization** — `ObjectInputStream.readObject`/`Serializable`, `NSKeyedUnarchiver` without `requiresSecureCoding`, Parcelables from untrusted intents.
- **Command/path injection** — `Runtime.exec`, file paths from intents/extras (traversal into provider or file writes).
- **Implicit-intent data**: validate all data arriving via intents/URL schemes — it is untrusted input (ties to ch06).

## XSS & WebView-adjacent (mostly covered by ch06)

Stored XSS only matters where content renders in the app's WebView with a bridge; reflected XSS is near-irrelevant (links open in browser). Don't inflate scanner XSS hits — see ch01 false-positive rules.

## Memory corruption

Native code (`lib/*.so`, NDK, iOS C/C++) is the only realistic memory-corruption surface — buffers from IPC/network/deserialization reaching `strcpy`/`memcpy`/`sprintf`, format strings, integer overflows. Verify binary protections below rather than hand-auditing every native function; use Ghidra/radare2 for review, fuzz JNI/native entry points where reachable.

## Binary / build protections (MASWE-0045)

Check with `checksec`/`rabin2 -I`/MobSF:
- **PIE/PIC** enabled (ASLR effectiveness), **stack canaries** (`__stack_chk_fail`), **ARC** (iOS), non-executable stack, RELRO on ELF.
- Android manifest: `android:debuggable="false"`, reasonable `minSdkVersion`/`targetSdkVersion`, modern APK signature scheme (v2+; v1-only/weak key = finding, see ch08).
- iOS: modern code signature format, no `get-task-allow` (debuggable entitlement) on release.

## Dependencies (MASWE-0044)

- Build the SBOM: `cdxgen`, MobSF, dependency-check/dependency-track; scan Gradle/CocoaPods/SPM artifacts.
- Match against known-vuln feeds; flag pinned-to-old versions of networking/crypto/serialization libs.
- Enforced updating: verify the app can force minimum-version upgrades (Play In-App Updates / App Store version check) — absence = MASWE-0043 for apps where lagging clients are dangerous.

## Test procedures

**Static:** MobSF full scan as baseline → manual grep of decompiled/plist/manifest: `debuggable`, SDK levels, signature scheme (`apksigner verify --verbose`), provider `query()` construction, `ObjectInputStream`, `DexClassLoader`/`NSBundle load`, dynamic-code paths, XML parser config, dependency manifests (`build.gradle`, `Podfile.lock`, `Package.resolved`).

**Dynamic:** probe exported providers with injection payloads via `content query`; fuzz intent extras and deep-link params reaching parsers; run vulnerable-dependency features and confirm the vulnerable code path is actually reachable (dependency present ≠ exploitable).

## Interpretation

- Dependency CVEs need reachability evidence — a vulnerable lib compiled in but never called is informational.
- Injection into providers/serialization is high when the entry point is exported or remote-driven (deep link, push payload, file import).
- Missing binary mitigations matter when native code parses untrusted input; cosmetic otherwise.
- Enforced-updating and platform-version findings are policy/maturity issues — report with business context, not as standalone "vulns" at L1.
