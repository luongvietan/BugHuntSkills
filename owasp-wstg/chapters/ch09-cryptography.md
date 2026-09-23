# 4.9 Weak Cryptography (WSTG-CRYP)

Source: WSTG v4.2 §4.9. TLS posture, padding oracles, plaintext channels, and algorithm hygiene. Most checks are passive (scanner + cert review); the padding oracle is the one deep exploit.

## Test list

- **WSTG-CRYP-01 — Weak Transport Layer Security.** Objective: validate TLS config and certificate strength. Procedure:
  - **Protocol/cipher review** (testssl.sh, SSLScan, nmap `ssl-enum-ciphers`, or testssl online): flag SSLv2 (DROWN), SSLv3 (POODLE), TLSv1.0 (BEAST), EXPORT suites (FREAK), NULL/anon ciphers, RC4 (NOMORE), CBC mode (BEAST/Lucky13), TLS compression (CRIME), weak DHE (LOGJAM). Compare against the Mozilla Server-Side TLS recommendations. Note: most of these need active-MitM resources — report real-world exploitability honestly.
  - **Certificate review**: key ≥2048-bit, signature ≥SHA-256 (no MD5/SHA-1), validity period (post-Sept-2020 certs max 398 days), trusted CA (public CA for external, internal CA acceptable internally — don't flag on that alone), SAN matching hostname (CN is ignored), wildcard scope concerns; check Issuer/SAN fields leaking internal names.
  - **Enforcement**: can pages/tokens still be fetched over HTTP? Is there a redirect-downgrade gap (no HSTS — see CONF-07)?
  - Interpretation: legacy protocol support or untrusted/weak certs = finding; rank by exploitability, not just presence.
- **WSTG-CRYP-02 — Padding oracle.** Objective: detect decrypt-oracle behavior on client-supplied ciphertext. Procedure: find candidates — random-looking values (often base-64-ish) whose decoded length is a multiple of 8 or 16 (block size, IV prepended: length = (blocks+1)×n); flip the last bit of the second-to-last block (byte at `len−n−1`), re-encode, send; repeat for earlier blocks; classify responses into three states — decrypts cleanly / decrypts garbled (app error) / padding error. Interpretation: three distinguishable states (error text, status, timing) = padding oracle → decrypt and forge plaintext without the key → session-state tampering, privesc. If responses look identical, the oracle likely isn't there.
- **WSTG-CRYP-03 — Sensitive info via unencrypted channels.** Objective: catch sensitive data traveling plaintext. Procedure: inventory what's sensitive for this app (auth data, session IDs, tokens, PII, card data); check every channel carrying it — HTTP pages, Basic auth over HTTP (`401` + `Authorization: Basic` header flow), form posts to `http://` actions, cookies without `Secure` that flow on HTTP requests; also check source/config/logs for hardcoded passwords and keys (`grep -rE` for pass/pwd/key patterns). Interpretation: any sensitive item on an unencrypted channel = finding; rule of thumb — if it's protected at rest it must be protected in transit.
- **WSTG-CRYP-04 — Weak encryption / hashing.** Objective: spot broken algorithms and bad parameters (mostly code/config review + scanner). Procedure: checklist — random unpredictable IV for AES (Java `SecureRandom`, not `Random`); ECC Curve25519 or RSA ≥2048 (PSS padding for RSA signatures); banned: MD5, RC4, DES, Blowfish, SHA-1, 1024-bit RSA/DSA, 160-bit ECDSA, 2-key 3DES; minimums — DH 2048, HMAC-SHA2, SHA-256, AES-128, PBKDF2/scrypt/bcrypt for passwords, ECDSA/ECDH 256; no CBC for SSH, no ECB anywhere; PBKDF2 iterations >10,000, never `PBKDF2WithHmacMD5`. Source greps: `MD4|MD5|RC4|RC2|DES|Blowfish|SHA-1|ECB`, Java `Cipher.getInstance`, `IvParameterSpec` reuse, `MessageDigest.getInstance("MD5")`, `Signature.getInstance("SHA1withRSA")`, plus hardcoded-key patterns. Interpretation: static scanners (Fortify/Coverity/etc.) and protocol scanners (Nessus/Nmap/OpenVAS) flag usage — verify the code path is actually reachable before reporting.

## Common findings

TLSv1.0/1.1 still enabled; CBC/RC4 suites offered; cert CN-only (no SAN) or expired; session cookie missing `Secure` on HTTPS-only app; padding-error differences in encrypted cookie/state blobs; MD5/SHA-1 password hashing; ECB mode in stored crypto; hardcoded keys in config.

## Escalation notes

Padding oracle → forge session state → privesc/ATO (SESS). Plaintext channels + network position = credential/session theft feeding SESS-09. Weak hashing → offline cracking of any leaked hashes. TLS gaps chain with CONF-07 HSTS absence and subdomain tricks (CONF-10). Keep PoCs passive — don't run active MitM against real users.
