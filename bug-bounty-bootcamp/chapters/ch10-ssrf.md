# Ch13: Server-Side Request Forgery

> **Metadata boundary:** cloud-metadata reach is read-only proof (harmless key/role name — never mint/use creds, `hacking-the-cloud` ch02/03 gates); internal-network sweeping needs explicit permission. SSRF = API7:2023 in the current taxonomy.

Source: Chapter 13. SSRF = make the *server* send a request to a destination you choose. Turns the server into a proxy: reach internal networks, cloud metadata, and localhost services invisible from the internet. Blind SSRF = no response body, only side effects (DNS/timing) — still exploitable.

## Hunting — where URL-fetching hides

- Obvious: `?url=`, `?image=`, `?callback=`, webhook config, "import from URL", RSS fetchers, PDF/thumbnail generators.
- Subtle: URL fields in XML (`<ImageUrl>`), `Referer` headers honored by the server, file-upload parsers that fetch embedded links (SVG `<image xlink:href>`, docx remote templates), API params that fetch resources, `Host` header tricks.
- Test: point at your own controlled endpoint — **Burp Collaborator**, interactsh, requestbin, or a `python -m http.server` on a public IP. Any inbound DNS/HTTP hit = confirmed SSRF even if response is invisible.

## What to hit once confirmed

- **Cloud metadata** (the crown jewels): `http://169.254.169.254/latest/meta-data/` (AWS — grab `iam/security-credentials/`), `http://metadata.google.internal` (GCP), `http://169.254.169.254/metadata/instance?api-version=…` (Azure, needs `Metadata: true` header → smuggle it if the app lets you control headers).
- **Internal network**: port-scan localhost (`http://127.0.0.1:22` → timing/error diff reveals open ports), internal hostnames from recon (`git.internal`, `admin.internal`), RFC1918 ranges (`10.`, `192.168.`, `172.16-31.`).
- **Localhost-only services**: admin panels, debug endpoints, unauthenticated DBs (Redis `gopher://`/`dict://` where the client supports it).

## Bypassing protections

- **Allowlist on hostname**: `attacker.com#trusted.com`, `trusted.com.attacker.com`, `trusted.com@attacker.com`, trailing dot `trusted.com.`, DNS that resolves to internal IP (rebind.it / your own domain A→`127.0.0.1`), URL-parser confusion (`http://trusted.com\@evil/`).
- **Blocklist on IPs**: alternate IP forms — `127.1`, `2130706433` (decimal), `0x7f000001` (hex), `0177.0.0.1` (octal), `127.0.0.1.nip.io`/`sslip.io`, IPv6 `[::1]`/`[::ffff:127.0.0.1]`.
- **Redirect escape**: allowlist validates your URL, your URL 302s to `169.254.169.254` — host a one-line redirector.
- **Scheme restrictions**: try `file:///etc/passwd`, `dict://`, `gopher://`, `sftp://`, `ldap://` — client lib often supports more than http.
- **DNS rebinding** for TOCTOU validators: domain resolves to public IP for the check, private IP for the fetch.

## Escalation & automation

Internal port scan → map internal services → hit metadata → steal temp IAM creds → (authorized env only) enumerate cloud. Blind SSRF → prove via DNS interaction + port-scan-by-timing. Escalate chains: SSRF → internal API → more SSRF (pivot); SSRF → read `file://` config → secrets → cloud takeover.

**First-SSRF checklist**: find URL-fetch features → Collaborator probe → map allowed schemes/hosts → internal IP/port probing → bypass families → metadata (test env) → document internal reach, not just "it fetched my URL".
