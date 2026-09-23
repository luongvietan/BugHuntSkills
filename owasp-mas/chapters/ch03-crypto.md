# Ch03: MASVS-CRYPTO — Cryptography

Source: MASTG crypto chapter, MASWE-0003/0007..0017, crypto tests, best-practices MASTG-BEST-0001/0005/0009/0025.

## Weakness types (MASWE)

| ID | Weakness | Core idea |
|---|---|---|
| MASWE-0007 | Improper encryption | Broken algorithms (DES/3DES/RC4/BLOWFISH), ECB mode, reused/predictable IVs, key misuse |
| MASWE-0008 | Improper hashing | MD5/SHA-1 for integrity; fast hashes (SHA-2) for passwords |
| MASWE-0009 | Improper MAC use | Wrong key handling, reuse, non-standard MAC construction |
| MASWE-0010 / 0011 | Improper signature generation / verification | Signing/verifying with wrong keys or accepting bad signatures |
| MASWE-0012 | Improper RNG | `java.util.Random`, `Math.random`, `rand()`, custom seeds for tokens/keys |
| MASWE-0013 | Improper key generation | Insufficient key size; keys derived from guessable material |
| MASWE-0014 | Improper key derivation | Password fed directly as key; missing KDF/iteration count |
| MASWE-0015 | Key rotation not implemented | Long-lived keys with no rotation path |
| MASWE-0016 | Key access not restricted | Keys usable without auth, exported, or readable |
| MASWE-0017 | Device secure lock not enforced | App proceeds without passcode/biometric-backed lock |
| MASWE-0003 | Keys outside platform keystore | Hardcoded or file-stored keys (overlaps STORAGE) |

## Common configuration failures (what to grep)

- **Broken primitives**: DES, 3DES, RC2, RC4, BLOWFISH, MD4, MD5, SHA-1, Dual_EC_DRBG, SHA1PRNG. Java: `Cipher.getInstance("DES")`, `"AES/ECB"`; `MessageDigest.getInstance("MD5"/"SHA-1")`. iOS: `CC_MD5`, `kCCAlgorithmDES`, deprecated `SecRandom` misuse, CommonCrypto DES/RC4 enums.
- **ECB mode** — identical plaintext blocks → identical ciphertext; patterns and replay. Expect `AES/GCM` (or CBC+HMAC) instead.
- **Hardcoded keys/IVs** — key literals in code/resources, IVs copied from library examples, IV derived from known data. Obfuscation does not fix this — dynamic instrumentation extracts them anyway.
- **Password-as-key** — user password used directly as AES key (short → padded, low entropy). Require PBKDF2/HKDF/Argon2 with proper iteration count and salt ≥ hash length.
- **Insecure RNG** — non-CSPRNG for keys/tokens/session IDs. Require CSPRNG (≥128-bit entropy): `SecureRandom`, `SecRandomCopyBytes`, `arc4random` family.
- **Key reuse across purposes** — same key for encryption + CBC-MAC, or one asymmetric pair for signature + encryption.
- **Custom crypto** — XOR/rolling "encryption", modified standard algorithms, homebrew formats. Encoding (Base64) is not encryption.
- **Key hygiene** — working keys/cipher state not cleared from memory after use; keys stored next to the data they encrypt.

## Test procedures

**Static:**
1. Inventory every crypto call: Android `javax.crypto.*`, `java.security.*`, `KeyStore`, spongycastle/BouncyCastle, Conscrypt; iOS `CommonCrypto`, `CryptoKit`, `SecKey*`, `Security.framework`.
2. Classify each by purpose (storage encryption, transport, integrity, KDF, RNG) → check algorithm, mode, key size, IV generation, key source against current standards.
3. Flag keys/IVs/salts in code or resources; check Keystore/Keychain usage and `setUserAuthenticationRequired`/Access Control flags.
4. Third-party/native: scan `lib/*.so` and frameworks for bundled OpenSSL/mbedTLS with weak defaults; check SBOM for old crypto libs.

**Dynamic:**
1. Hook crypto APIs with Frida (`Cipher.doFinal`, `CCCrypt`, `SecItemCopyMatching`) → log algorithm/mode/key/IV/plaintext at runtime; catches crypto config invisible to static review and keys assembled at runtime.
2. Randomness: collect generated tokens/IVs — repeats or short/sequential values = weak RNG evidence.
3. Determinism probe: encrypt the same plaintext twice via the app — identical ciphertext → ECB or fixed IV.
4. TLS key checks overlap ch05 (cert pinning, two-way TLS client-cert password storage).

## Interpretation (current guidance)

- Acceptable today: AES-GCM-256 or ChaCha20-Poly1305; SHA-256/384/512, SHA-3, BLAKE3; RSA ≥3072, ECDSA P-384, EdDSA — Ed25519 is the mobile norm (Ed448 exists but is rare on-device); RSA/DH ≥3072 or ECDH P-384 for key establishment. PBKDF2 at the current OWASP floor — ~210k–600k iterations depending on the hash (e.g., ~600k for HMAC-SHA256, ~210k for HMAC-SHA512; verify the latest OWASP Password Storage Cheat Sheet) — or PHC algorithms such as Argon2id.
- Verify **purpose-fit**: SHA-256 hashing a screen-resolution analytics value is fine; MD5 for integrity or SHA-256 for passwords is not. Non-security contexts downgrade or void the finding.
- Padding-oracle surface: CBC without MAC + distinguishable decryption errors — confirm error side channels before claiming.
- Post-quantum note: NIST FIPS 203/204/205 (ML-KEM/ML-DSA/SLH-DSA) are the forward path; flag "future-proof" claims using unreviewed PQ schemes.
- Keys must live in Keystore/Keychain/Secure Enclave where available; symmetric keys stored beside ciphertext = MASWE-0003 finding.
