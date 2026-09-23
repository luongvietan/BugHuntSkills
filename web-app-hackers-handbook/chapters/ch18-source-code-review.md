# Ch19: Finding Vulnerabilities in Source Code

Source: Chapter 19. White-box complements black-box: code review finds what dynamic testing can't see (second-order paths, logic subtleties, race conditions), and knowing the *signatures* makes you a faster black-box hunter too.

## Approaches

- **Black-box ↔ white-box hybrid**: use dynamic findings to target review (that param's handler), and use review findings to craft dynamic probes (the hidden param's exact name). Either alone is incomplete.
- **Two reading strategies**: (a) **trace taint forward** — map every user-input source and follow it to sinks; (b) **search dangerous-API backward** — grep for sink APIs then verify reachability from input. Combine: grep gives coverage, tracing gives context.

## Vulnerability signatures (grep targets)

- **XSS**: any request-data expression reaching output/response-write without encoding for the target context (HTML vs attribute vs JS vs URL); stored sources (DB fields, files) reaching display.
- **SQL injection**: string concatenation of request data into query text — contrast with parameterized/prepared calls; second-order: DB/file/session data re-entering a query.
- **Path traversal**: request data in file-path construction (`open`, `include`, `read`, `File`, `fopen`) — check for `..`/canonicalization validation *and its order*.
- **Arbitrary redirect**: request data in `Location`/redirect calls — unvalidated = open redirect.
- **OS command injection**: request data reaching exec/spawn/system APIs.
- **Backdoors**: hardcoded creds, magic params (`?test=`, `debug`, secret headers), commented-out auth checks.
- **Native bugs** (C/C++ components): unbounded copies (`strcpy`, `sprintf`, `gets`), length arithmetics feeding allocations/copies (integer/overflow/signedness), `printf(var)` format-string sites.
- **Logic smells**: comments revealing assumptions ("user will have…"), presence-checks (`if (param != null) → trusted`), static/shared fields holding per-user data, single-pass sanitizers, validation done before a decode/transform.
- **Source comments**: TODO/FIXME/hack notes flag known-broken spots; commented-out security checks; author names → git blame context.

## Platform-specific input maps

- **Java**: request sources — `getParameter`, `getQueryString`, `getHeader`, cookies, `getInputStream`/`ServletInputStream`; session — `get/setAttribute`; dangerous sinks — `File`, `FileInput/OutputStream`, `Runtime.exec`, `Statement.execute*`, `sendRedirect`, dynamic `Class.forName`/reflection/`Method.invoke`, script-engine `eval`.
- **ASP.NET/C#**: `Request.Params/QueryString/Form/Cookies`, `HttpMethod`, `Request[...]` (merged sources!), `Request.Url`; `Session[]`; sinks — `File.*`, `Process.Start`, `SqlCommand` concat, `Response.Redirect`, `Server.Execute/Transfer`, `HttpResponse.Write`, `eval`-ish code providers, serialization (`BinaryFormatter` = object injection).
- **PHP**: `$_GET/$_POST/$_COOKIE/$_REQUEST/$_SERVER`, `php://input`; sinks — `include/require` (LFI/RFI), `eval`/`preg_replace /e`, `system/exec/shell_exec/popen`, `mysql_query` concat, `unserialize`, `file_get_contents` with wrappers.
- **Classic ASP**: `Request.QueryString/Form/Cookies`, `Request.BinaryRead`; sinks — `Server.Execute`, `Execute/Eval`, `ADODB` concat, `Scripting.FileSystemObject`, `WScript.Shell`.

## Review method

1. Map the request-routing & dispatch (front controller, filters, handlers) — know which code runs per request.
2. Identify every input source per platform (incl. headers, cookies, file names, multipart fields, deserialized objects).
3. Grep the sink list; for each hit, trace backward to an input source; note every transform en route (decode? escape? truncate? — ordering = bypass, see ch01).
4. Review cross-component assumptions: session vs static scope, tier-to-tier trust, per-request vs per-user storage (ch10 logic patterns in code form).
5. Environment/config review: framework settings, debug flags, error verbosity, safe-mode equivalents.

## Checklist

- [ ] All input sources enumerated for the platform.
- [ ] Dangerous-sink grep run; every reachable sink traced to a source.
- [ ] Transform-ordering reviewed at each boundary (decode/escape/validate order).
- [ ] Session-vs-static and per-user storage audited.
- [ ] Comments/backdoors/debug flags searched.
- [ ] Findings converted to dynamic PoCs (and vice versa).
