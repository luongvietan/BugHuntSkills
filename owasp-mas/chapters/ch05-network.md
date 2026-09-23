# Ch05: MASVS-NETWORK — Network Communication

Source: MASTG network chapter (TLS, trust stores, pinning, MITM), MASWE-0026..0028, network tests, best-practices MASTG-BEST-0020/0042/0043.

## Weakness types (MASWE)

| ID | Weakness | Core idea |
|---|---|---|
| MASWE-0026 | Network traffic not encrypted | Cleartext HTTP, weak TLS versions/ciphers, ATS/NSC exceptions |
| MASWE-0027 | Insecure certificate validation | Accept-all TrustManagers/hostname verifiers, SSL error overrides, trusting user-added CAs |
| MASWE-0028 | Insecure identity pinning | Missing/expired/bypassable pinning (MAS-L2 requirement) |

## Test procedures

**Cleartext & TLS version (MASWE-0026):**
- Static: Android `networkSecurityConfig` (`cleartextTrafficPermitted`, per-domain overrides), manifest `usesCleartextTraffic`; iOS `Info.plist` `NSAppTransportSecurity` (`NSAllowsArbitraryLoads`, per-domain `NSExceptionAllowsInsecureHTTPLoads`, weak `NSExceptionMinimumTLSVersion`); grep hardcoded `http://` URLs; cross-platform frameworks (Flutter/React Native/Xamarin) have their own network stacks — check their config too.
- Dynamic: proxy all traffic (Burp/ZAP/mitmproxy) and sniff (`tcpdump`/Wireshark/bettercap) — cleartext observed on the wire = fail regardless of config intent. Watch `Socket`/`URLSession`/`Network.framework` calls that bypass HTTP stacks (custom TLS or none).
- Baseline: TLS 1.2/1.3 only; SSL and TLS 1.0/1.1 deprecated. Verify server suites with testssl.sh/nscurl; cipher names read `Protocol_Kx_WITH_Cipher_MAC` (e.g., `TLS_RSA_WITH_3DES_EDE_CBC_SHA` = bad on every axis).

**Certificate validation (MASWE-0027):**
- Static: hunt for accept-everything code — Android `X509TrustManager` with empty `checkServerTrusted`, `ALLOW_ALL_HOSTNAME_VERIFIER`, `HostnameVerifier` returning true, `WebViewClient.onReceivedSslError → handler.proceed()`; iOS `URLSessionDelegate`/`WKNavigationDelegate` calling the completion handler with `.useCredential`/accepting challenges unconditionally, `SecTrustEvaluate` misuse.
- Trust-store scope: Android NSC or iOS trusting **user-added CAs** lets any installed cert MITM the app — check `trust-anchors` (`<certificates src="user">`) and target SDK (apps targeting API ≥24 distrust user CAs by default; overrides are the finding).
- Dynamic: present a self-signed/rogue-CA cert through the proxy — connection succeeding = validation broken. GMS/Conscrypt security provider outdated on old Android = weak TLS stack.

**Identity pinning (MASWE-0028):**
- Static: NSC `<pin-set>` (SPKI `subjectPublicKeyInfo` hashes, expiry attr, backup pin), OkHttp `CertificatePinner`, iOS ATS `TSKConfiguration`/TrustKit, manual `SecTrust` pin checks.
- MASTG position: pin at development time, pin SPKI hash, only to endpoints you control, always ship a backup pin + update path. Missing pinning is *not* a vuln at MAS-L1 — it is required at MAS-L2. Pinning protects against compromised/malicious CAs, not against an attacker controlling the device.
- Dynamic: expired pins, missing backup pin, or proxy cert still accepted → pinning ineffective. To *bypass* pinning for your own testing see ch09 (Frida/Objection SSL-unpinning scripts, SSL Kill Switch).

## MITM positioning (how to get the traffic)

| Layer | Method | Tools |
|---|---|---|
| App | Hook network APIs (`HttpUrlConnection`, `NSURLSession`) | Frida |
| TLS | Hook `SSL_read`/`SSL_write` | Frida, SSL Kill Switch |
| Proxy | System proxy the app respects | Burp, ZAP, mitmproxy |
| Packet | Sniff all TCP/UDP (no decryption) | tcpdump, Wireshark |
| Network | ARP spoof, rogue AP | bettercap, hostapd |

Non-HTTP traffic: MITM Relay / Nope-Proxy for raw TCP; Flutter ignores system proxy + pins — use reFlutter/disable-flutter-tls-verification or hook `ssl_cert_verification`.

## Interpretation

- Any sensitive data in cleartext = reportable regardless of pinning debates.
- Broken validation (accept-all TrustManager, `proceed()` on SSL error, user-CA trust for sensitive apps) is the high-severity core of this category — it silently voids all TLS.
- Pinning absence: informational at L1, required control at L2; expired pins/no backup pin = operational failure risk.
- Evaluate the server endpoint too: TLS terminates at a proxy/load balancer — weak config there still exposes app data.
