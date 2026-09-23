# Ch04: MASVS-AUTH — Authentication & Session Management

Source: MASTG authentication chapter (stateful/stateless, OAuth2, 2FA, logout, device blocking), MASWE-0017..0025, local-auth tests, best-practices MASTG-BEST-0031/0036/0037/0038.

## Weakness types (MASWE)

| ID | Weakness | Core idea |
|---|---|---|
| MASWE-0018 | No authN/authZ on app components | Exported components/IPC callable without permission checks |
| MASWE-0019 | No autofill/credential-provider support | Password fields defeat password managers → weak passwords |
| MASWE-0020 | Local authentication bypassable | Biometric prompt returns success without crypto binding — hookable |
| MASWE-0021 | Fallback to non-biometric allowed | Device-credential fallback (PIN) for sensitive transactions weakens control |
| MASWE-0022 | Keys not invalidated on enrollment change | New fingerprint added → old key still signs = biometric bypass persistence |
| MASWE-0023 | No step-up auth for sensitive actions | Transfers/exports don't re-authenticate |
| MASWE-0024 | Data accessible after session ends | Post-logout screens/cache/tokens still usable |
| MASWE-0025 | No non-repudiation | Critical actions unverifiable |
| MASWE-0017 | Device lock not enforced | App usable on devices with no passcode |

## Architecture review (do this first)

- **Stateful sessions**: server-issued opaque session ID. Verify: random high-entropy server-side generation, HTTPS-only exchange, not persisted to disk on device, validated per privileged request, server-side invalidation on logout/timeout.
- **Stateless tokens (JWT)**: decode every token — check `alg` enforced (reject `none`, reject RS256→HS256 confusion), `exp` honored server-side, `jti` for replay, `aud` for cross-service relay, no PII/sensitive claims in payload, signature verified on every request, tokens stored in Keychain/Keystore not files. Test with Burp JWT extensions.
- **OAuth2 on mobile**: authorization-code flow **with PKCE** only (implicit is deprecated); redirect via claimed https scheme (universal links/App Links) not hijackable custom schemes; `state` required (CSRF); auth done in system browser (`ASWebAuthenticationSession`/Custom Tabs), not embedded WebView (credential phishing + no cookie sharing); client secret must not ship in the app.
- **Password policy**: verify server-side policy; mobile side needs autofill/password-manager support (autofill hints, `textContentType`) or users pick weaker passwords.

## Local (on-device) authentication — the mobile-specific surface

- **Crypto-bound vs UI-bound biometrics**: Android `BiometricPrompt` with a CryptoObject (keystore key unlocked by auth) vs iOS `LAContext.evaluatePolicy` returning a boolean. Boolean-only flows are bypassable with a Frida hook that returns success — always prefer crypto-bound design.
- **Key invalidation**: keys must be created with `setInvalidatedByBiometricEnrollment(true)` / iOS equivalent so a newly enrolled fingerprint/face can't unlock old data.
- **Explicit confirmation**: biometric auth for sensitive ops should require explicit user action (not passive), and high-value actions need step-up auth.
- **Fallback**: `setAllowedAuthenticators(BIOMETRIC_WEAK|DEVICE_CREDENTIAL)` or `LAPolicyDeviceOwnerAuthentication` lets a device PIN substitute for biometrics — acceptable for convenience, risky for sensitive transactions (MASWE-0021).
- **No-locked-device check**: app should detect a device with no secure lock and refuse sensitive functions.

## Test procedures

**Static:**
1. Grep biometric APIs: Android `BiometricPrompt`, `canAuthenticate`, `KeyGenParameterSpec` (check `setUserAuthenticationRequired`, `setInvalidatedByBiometricEnrollment`, validity duration); iOS `LAContext`, `LAPolicy`, `SecAccessControl`, `kSecAttrAccessibleWhenPasscodeSetThisDeviceOnly`.
2. Identify whether auth result merely gates UI or unlocks a keystore key (crypto binding = the presence of a CryptoObject/`SecItemCopyMatching` gated call).
3. Session code: where tokens/IDs are written (prefs/plist vs Keystore/Keychain), logout handlers (server call + local purge), timeout logic.
4. OAuth: PKCE params in requests, `state`, redirect scheme, embedded vs system browser.

**Dynamic:**
1. Bypass probe: hook the auth callback (objection `ios ui biometrics_bypass` / Frida `BiometricPrompt` result override) — if a boolean flip grants access, crypto binding is absent (MASWE-0020 confirmed).
2. Enrollment-change test: enroll a new fingerprint/face → previously protected data/keys must become unusable (MASWE-0022).
3. Replay & expiry: replay captured tokens; wait past `exp`/timeout and retry; logout → reuse old token and re-open cached screens (MASWE-0024).
4. JWT attacks: `alg:none`, strip signature, RS256→HS256 with public key, weak-secret crack; verify rejection server-side.
5. OAuth intercept: attempt redirect to a claimed custom scheme/host; drop/modify `state`.
6. Rate limiting on PIN/OTP endpoints, SMS-OTP weaknesses (SIM swap risk noted; prefer TOTP/push/PKI transaction signing for high value).
7. Login-activity/device blocking: excessive-failure lockout and notification of new-device login.

## Interpretation

- **UI-bound biometrics** (boolean callback only) = bypassable; report with the Frida hook as PoC. Crypto-bound designs resist this — the hook can't unlock the keystore key.
- Session findings live mostly server-side — the MASTG defers to OWASP WSTG for auth/session depth; mobile adds token storage, biometric binding, and OAuth-on-device specifics.
- Severity: local-auth bypass on a banking app > on a casual app; always tie to what the bypass unlocks.
- 2FA: SMS-OTP is the weak baseline (interception/SIM swap); confirm step-up auth exists for funds/sensitive-data actions.
