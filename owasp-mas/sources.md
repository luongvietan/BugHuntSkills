# sources.md — owasp-mas

## Sources

- **Primary**: OWASP MASTG v2.0.0 — https://mas.owasp.org — CC-BY-SA 4.0,
  companion to the MASVS verification standard. Rolling release — chapter
  content here reflects the v2.0.0 snapshot plus marked refresh callouts.
- **Standard**: OWASP MASVS (mas.owasp.org/MASVS) — control categories
  STORAGE / CRYPTO / AUTH / NETWORK / PLATFORM / CODE / RESILIENCE; upstream
  now also ships a **MASVS-PRIVACY** category (not yet covered here).
- Methods-refresh source register: `docs/superpowers/2026-09-23-bug-bounty-source-register.md`
  (BugBounty workspace) — pins v2.0.0 status and rolling-release note.

## Provenance & currency policy

- Chapters synthesized from MASTG v2.0.0; MASWE/test IDs preserved verbatim.
- Tool invocations are point-in-time: Frida/Objection/MobSF/apktool/jadx/
  idevicebackup2 flags drift — verify against current releases; prior
  refresh fixes recorded (PBKDF2 floor, idevicebackup2 `--full` removal,
  DexClassLoader grep).
- Backend/API testing of the mobile app's server side routes to Task-4
  skills (`hacking-apis`, `owasp-api-security-top-10`), not this skill.

## Edge policy (testing)

- Test devices carry test data only — emulator or wiped device + dedicated
  account; a rooted device holding real user data is not a test device.
- Root/jailbreak-required findings count only when the program includes
  them in scope.

## Review history

- 2026-09-23: version status verified (v2.0.0, rolling), MASVS-PRIVACY gap
  noted, device-rules + tool-currency callouts, sources.md created
  (methods-refresh T6).
