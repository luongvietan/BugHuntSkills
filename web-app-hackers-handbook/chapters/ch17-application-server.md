# Ch18: Attacking the Application Server

Source: Chapter 18. The server under the app is its own attack surface: default content, misconfig, proxy behavior, framework flaws, and the WAF that pretends to fix the app. Often the easiest wins live here.

## Vulnerable configuration

- **Default credentials**: admin consoles (Tomcat manager, JBoss jmx-console, WebLogic, db consoles) with vendor defaults — fingerprint the product then try its *known* defaults (tomcat/tomcat, admin/admin…). Consoles on nonstandard ports/paths are still reachable.
- **Default content**: sample apps, test pages, docs, sample scripts (often with known bugs), admin interfaces shipped enabled. Enumerate product-specific default paths after fingerprinting (ch03).
- **Debug functionality**: debug servlets (`/test`, diagnostics, invoker servlets — run any class), status pages, tracing — map then probe.
- **Powerful functions**: exposed admin ops — upload-deploy, JMX/invoker calls, restart, config view; often unauthenticated on internal paths.
- **Directory listings**: on where it shouldn't be → file inventory, backup/config files. Request dir without index; watch for `403 vs 404` (exists-but-listed vs absent).
- **WebDAV**: `OPTIONS`, `PROPFIND`, `PUT`, `DELETE`, `MOVE`, `SEARCH` — if enabled on content dirs → upload/modify files. Probe methods on every path; try auth bypass via method confusion (`PUT` where `GET` is checked).
- **Server as proxy**: open-proxy behavior (`CONNECT`, absolute-URI GET) → hide origin, reach intranet, hit the server itself loopback. `Allow`/forwarded-header tricks reach admin resources.
- **Virtual hosting**: default vhost vs named vhosts; `Host` manipulation, IP literal, missing Host → reach other tenants/default admin areas (see ch16).

## Vulnerable server software & frameworks

- **Memory bugs in the platform**: historical overflows in server software, WebDAV handling, encoding/canonicalization bugs (IIS Unicode/double-decode traversal — the class recurs: *decode-order bugs in the server itself*). Old versions = catalogued CVEs → fingerprint precisely (banner, error, behavior quirks like `/:` vs `/` handling).
- **Framework flaws**: default framework behaviors (ViewState, debug handlers, servlet invokers, .htaccess/equivalent processing), framework-level auth bypasses, and *server-level* traversal/`..` quirks distinct from the app's own.

## Finding & evading

- **Methodology**: fingerprint product+version → enumerate its default paths/creds/content → probe config weaknesses (methods, listings, proxy, vhosts) → check versioned CVEs → test canonicalization quirks at the *server* layer (overlong encodings, `..;/`, double-decode, `::` tricks) — these differ from app-layer handling.
- **WAFs**: identify presence (provoke a block on a blatant probe, note the response signature). Evasion mirrors ch11's catalog at WAF scale: encodings the WAF doesn't normalize (double-URL, multibyte, case, comments), NULL bytes truncating native-code match, path/method/content-type it doesn't inspect (POST body vs query, multipart), and splitting payloads across parameters (HPP, ch09). WAF rules often cover only known attack strings — novel but equivalent syntax sails through. A WAF finding is reportable both as bypass *and* as the app having no real defense.

## Checklist

- [ ] Product+version fingerprinted; vendor defaults (creds, paths, content) tried.
- [ ] Admin/debug/status interfaces probed on all ports/paths.
- [ ] `OPTIONS`/WebDAV methods enumerated per directory.
- [ ] Proxy behavior (CONNECT, absolute URI) and Host-header/vhost handling tested.
- [ ] Server-layer canonicalization quirks tried (separate from app layer).
- [ ] WAF presence/rules characterized; bypass set attempted.
