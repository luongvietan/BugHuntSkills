# Ch5: Tactical Fuzzing — XSS & SQLi

> **Ceiling note:** polyglots and error-based probes are detection payloads — confirmation ends the test (`alert(document.domain)`, sleep); extraction is report narrative. Payload families: `payloads-all-the-things` ch01/02.

Source: `05_XSS` + `06_SQLi`. Tactical fuzzing = the 80/20 of input testing: for time-sensitive hunting, lead with multi-context **polyglot** payloads, then escalate to targeted payloads only where the polyglot lands. Per feature, ask the core question first.

## XSS — "does the page functionality display something to the user?"

Polyglot strategy: one string that survives multiple contexts (attribute, element, JS string, comment) so a single request tests them all. TBHM collects three:

- **Rsnake's polyglot** — long multi-context string mixing quote/comment closers with script/img payloads; fires across HTML + JS contexts (XSS cheat-sheet heritage).
- **Ashar Javed's polyglot** — chains `marquee`/`img onerror`/`plaintext`/`isindex formaction` variants to defeat filters that only strip `<script>`.
- **Mathias Karlsson's compact polyglot** — short enough to remember:

```
" onclick=alert(1)//<button ' onclick=alert(1)//> */ alert(1)//
```

Use polyglots as smoke tests on search/registration/contact/reset/comment forms (the n-minute assessment in ch08 does exactly this); when one fires, map the actual reflection context and craft the minimal clean payload for the report.

### XSS input vectors (beyond obvious form fields)

- Customizable themes/profiles via CSS
- Event or meeting names (stored, viewed by others)
- URI-based reflection (path, query, fragment)
- Content imported from third parties (e.g., social-platform integrations)
- JSON POST bodies — check the **returned content-type**; a JSON body reflected as `text/html` is XSS-able
- File-upload filenames, and uploaded files themselves (HTML/SWF rendered same-origin)
- Custom error pages echoing bad input
- **Fake params** — append nonexistent params carrying payloads (`?realparam=1&foo=bar'`) — many apps reflect unknown inputs
- Login and forgot-password forms (self-XSS → chain candidates)

### SWF parameter XSS (legacy but still on old stacks)

Flash params like `onload`, `allowedDomain`, `movieplayer`, `xmlPath`, `eventhandler`, `callback` took JS-injection strings of the `}catch(e){alert(...)}//` family — full param list on the OWASP Flash XSS page. On modern stacks the same pattern lives in **JS-component params and postMessage handlers** — grep pages/JS for handler names passed via URL.

## SQLi — "does the page look like it calls on stored data?"

- **Karlsson SQLi polyglot** — works in single-quote, double-quote, and straight-into-query contexts:

```
SLEEP(1) /*' or SLEEP(1) or '" or SLEEP(1) or "*/
```

- **Blind is predominant; error-based is rare.** Time/benchmark payloads are the workhorse — e.g. MySQL heavy-query probes like `' + BENCHMARK(40000000,SHA1(1337)) + '` (URL-encode as needed) to force a measurable delay without output.
- **Fuzz lists**: SecLists carries dedicated SQLi fuzz/payload lists (incl. Haddix's own LFI/SQLi lists).
- **Lots of injection in web services** — SOAP/XML/JSON API params get less scrutiny than HTML forms.

### SQLMap is king — TBHM workflow

- Feed it a Burp log (`sqlmap -l burp.log`) or saved request (`-r req.txt`) — instrument straight from the proxy via the SQLiPy plugin.
- Use **tamper scripts** when a blacklist/WAF filters raw payloads.
- Confirm blind hits with time deltas; keep PoC minimal (`version()`, current user) — never dump tables to prove impact.

### Per-DBMS cheat sheets (from TBHM's resource list)

MySQL, MSSQL, Oracle, PostgreSQL, Access, Ingres, DB2, Informix, SQLite, Rails/ActiveRecord — PentestMonkey cheat sheets, Reiners' MySQL filter-evasion sheet, EvilSQL MSSQL error/union/blind sheet, rails-sqli.org. Pair with `payloads-all-the-things` for the modern equivalent.
