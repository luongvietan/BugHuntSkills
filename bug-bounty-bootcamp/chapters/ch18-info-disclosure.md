# Ch21: Information Disclosure

Source: Chapter 21. Info leaks = data exposed that aids attack: credentials, internal IPs, source code, stack traces, directory listings, config backups, PII. Alone = low sev; chained = breach. The skill: systematic leak-hunting + *validating* what leaked.

## Hunting surface

- **Path traversal**: `?file=../../etc/passwd`, encodings (`..%2f`, `%2e%2e`, `....//`, `..\\`), absolute paths, `php://filter` for source.
- **Wayback Machine**: old endpoints, removed-but-live pages, historical JS with old API keys, archived directory listings (`waybackurls`, gau).
- **Search engines & dorking**: `site:target.com ext:log|env|sql|conf`, `intitle:"index of"`, GHDB queries; Paste sites + public gists for leaked creds/source.
- **Exposed `.git`**: `/.git/config` 200? → dump with GitTools/git-dumper → full source history → secrets, endpoints, code review (Ch22) ammo.
- **Source/JS inspection**: source maps (`.map` files → original source), comments, hardcoded keys in JS bundles, internal endpoints, debug flags, feature flags revealing admin paths.
- **Config & backup files**: `web.config~`, `.env`, `config.php.bak`, `.DS_Store`, `wp-config.php~`, `.svn/`, `.hg/` — found via directory brute-force with extension wordlists.
- **Error messages**: verbose errors → stack traces, SQL fragments, file paths, versions. Force them: malformed input, type confusion, oversized input.
- **API over-sharing**: endpoints return more than the UI shows (the book's case: profile API leaks private API token → impersonation). Always diff raw response vs rendered page.
- **Metadata**: EXIF in uploaded images, PDF metadata, office doc properties → usernames, internal paths.

## Validation (the step that makes it a bug)

- Leaked credential: **verify it's current** — test only against in-scope login on your own test account where allowed; a dead key is informational.
- Internal IP/hostname → feed to SSRF testing (ch10).
- Version strings → match CVEs → known-exploit path.
- Source code → code review for vulns (ch19).
- PII records → quantify scope (count, field types) without copying data.

## Escalation & reporting

Chain math: `.git` → source → creds → cloud; stack trace → path → LFI; version → CVE → RCE. Report = what leaked + why it's sensitive + the verified impact, not just "info disclosure at /X".

**Checklist**: traversal probes → archive/paste sweep → `.git` & backups → JS/source review → error fuzzing → validate each secret → chain into real impact.
