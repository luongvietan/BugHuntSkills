# Ch02: MASVS-STORAGE — Data at Rest

Source: MASTG Android/iOS storage chapters, MASWE-0001..0006 (plus MASWE-0036 keyboard caching overlap), MASTG-TEST storage cases, best-practices MASTG-BEST-0004/0023/0024/0026.

## Weakness types (MASWE)

| ID | Weakness | Core idea |
|---|---|---|
| MASWE-0001 | Sensitive data stored unencrypted in private storage | Sandbox files/DBs readable on rooted/jailbroken or extracted devices |
| MASWE-0002 | Sensitive data stored unencrypted outside private storage | External storage / shared containers / App Groups readable by other apps |
| MASWE-0003 | Crypto keys stored outside the platform keystore | Keys in code, files, prefs instead of Keystore/Keychain/Secure Enclave |
| MASWE-0004 | Sensitive data hardcoded in the app package | Secrets compiled into resources/dex/binary — trivially extractable |
| MASWE-0005 | Insertion of sensitive data into logs | Logcat/NSLog/os_log persists beyond the session, readable by tooling |
| MASWE-0006 | Sensitive data not excluded from backup | Android Auto-Backup / iTunes-iCloud backup carries secrets off-device |
| MASWE-0036 | UI exposure incl. keyboard caching | Text-field input cached in keyboard/autocorrect dictionaries |

## Where Android apps leak

- `SharedPreferences` XML, SQLite/Room/DataStore files under `/data/data/<pkg>` — plaintext unless EncryptedSharedPreferences/SQLCipher or Keystore-wrapped keys.
- **External storage** (`/sdcard`, `getExternalFilesDir`) — world-accessible; treat anything written there as public.
- **Logs**: `Log.d/v/i` survive in logcat (`adb logcat`, pidcat); release builds must strip debug logging.
- **Backups**: `android:allowBackup="true"` + Auto-Backup rules; `adb backup`/`bmgr` or `backup_exclude` rules decide what leaves the device.
- **Keyboard cache & screenshots**: autocorrect learns typed secrets; app-switcher snapshots leak screen content (see ch06).

## Where iOS apps leak

- **UserDefaults (plist)**, files in `Documents/`, `Library/Application Support`, `Caches/`, `tmp/` — plaintext by default; only Data Protection classes encrypt, and only while locked.
- **Keychain mis-scope**: items without proper `kSecAttrAccessible` (e.g., `...Always`) or missing biometric Access Control.
- **Logs**: `NSLog`/`print`/`os_log` visible via Console/`idevicesyslog`.
- **Backups**: files not flagged `NSURLIsExcludedFromBackupKey` end up in iTunes/iCloud backups; App Group containers share data with extensions/other apps.
- **Snapshots & pasteboard**: background screenshots in `Library/Caches/Snapshots`; general pasteboard readable by any app (see ch06).

## Test procedures

**Static (references tests):**
1. Android: decompile (`jadx`/apktool) → grep storage APIs: `SharedPreferences`, `openFileOutput`, `SQLiteDatabase`, Room `@Entity`, `DataStore`, `getExternalFilesDir`, `MODE_WORLD_*`, `Log.` calls, manifest `allowBackup`/backup rules.
2. iOS: `Info.plist` ATS/privacy keys, entitlements (App Groups), grep `UserDefaults`, `NSFileManager` writes, `write(toFile:)`, `NSLog`, `os_log`, Keychain `SecItemAdd` accessibility flags, `isExcludedFromBackup`.
3. Hunt hardcoded secrets in resources/`strings.xml`/assets/plist and code (gitleaks, Apkleaks, regex over decompiled source).

**Dynamic (runtime tests):**
1. Install → exercise every flow entering sensitive data → snapshot app data dirs before/after; diff for new files; inspect contents for plaintext secrets.
2. Android: `adb logcat`/`pidcat` while exercising; iOS: `idevicesyslog`/Console. Search output for tokens, PII, passwords.
3. Android external storage: list `/sdcard` and `Android/data` before/after — any new files must be checked for sensitive content.
4. Backup: Android `adb backup -f app.ab <pkg>` (or `bmgr`); iOS unencrypted iTunes backup → inspect with iOSbackup/idevicebackup2 — sensitive files present = fail.
5. Keyboard: type a secret into inputs → check `UserDictionary`/keyboard cache files / custom-keyboard capture; iOS: verify `isSecureTextEntry`/`autocorrectionType` on sensitive fields.
6. iOS: pull `Library/Caches/Snapshots` after backgrounding on a sensitive screen; check pasteboard contents after copying.

## Tools

`adb` (pull/run-as/logcat/backup), apktool + jadx (decompile), MobSF (automated storage findings), objection (`env`, `ios nsuserdefaults get`, `android hooking`), Frida (trace storage APIs at runtime), Fridump (memory), iOS: Keychain-Dumper, libimobiledevice (`idevicebackup2`, `idevicesyslog`), iOSbackup, Filza; secrets: gitleaks, Apkleaks.

## Interpretation

- **Fail** when a file/log/backup stores *defined* sensitive data unencrypted, or lands outside private storage, or keys sit outside Keystore/Keychain.
- Sandbox plaintext alone is weak-moderate (needs root/physical access) — severity scales with data sensitivity and whether encryption/keystore was expected (MAS-L2).
- Hardcoded secrets in the package are *always* reportable — extraction needs no device compromise.
- Verify context: logging a non-sensitive analytics value is not a leak; confirm the data matches the agreed sensitive-data list before reporting.
- Remediation anchors: Android Keystore + EncryptedSharedPreferences/Tink; iOS Keychain + Data Protection classes; exclude from backups; strip logging in release.
