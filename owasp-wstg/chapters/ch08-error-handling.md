# 4.8 Error Handling (WSTG-ERRH)

Source: WSTG v4.2 §4.8. Small category (WSTG-ERRH-02 stack traces merged into ERRH-01): what the system reveals when it fails — and whether failure can be weaponized for recon or DoS.

## Test list

- **WSTG-ERRH-01 — Improper error handling (includes stack traces, ERRH-02).** Objective: provoke errors and mine what they disclose. Procedure:
  - **Web server layer**: request nonexistent paths (404s), existing directories (403 vs blank vs *directory listing*), and malformed HTTP — huge URI, broken header format, wrong protocol version — to surface default server error pages that bypass app-level handlers.
  - **Application layer**: identify input points and expected types; fuzz with type-mismatched and structural payloads (a stray `}`/`]` in JSON bodies, oversized text in short fields, CRLF into parsed params, filename-illegal chars); watch for stack traces, memory dumps, mishandled exceptions, and verbose framework errors.
  - Interpretation: errors leak internal APIs, framework/language versions, file paths, query fragments, and service topology — direct recon for attack chaining (INFO-08 fingerprinting, INPV-05 SQLi error pages, ERRH → padding-oracle side channels in CRYP-02). Also test DoS potential: can an unhandled exception deadlock or panic the app?

## Common findings

Default IIS/Apache/nginx error pages (version + tech); framework stack traces naming internal classes/files/line numbers; SQL error text revealing query structure; different error behavior between edge and app layers; verbose JSON/XML parse errors; debug mode left enabled.

## Escalation notes

Error output is rarely the final finding — it's the map: stack traces → exact framework version → targeted CVEs (INFO-02/08); DB errors → SQLi confirmation and column discovery (INPV-05); file paths in errors → traversal targets (ATHZ-01); internal hostnames/APIs → SSRF targets (INPV-19). Report the mechanism plus what it enables.
