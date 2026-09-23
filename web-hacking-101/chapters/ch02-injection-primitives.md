# Ch 2 — HTML Injection, HTTP Parameter Pollution, CRLF

Three "first-order injection" classes: your input lands unmodified in markup, in a forwarded request, or in response headers.

## HTML Injection ("virtual defacement")

HTML input rendered as-is — distinct from XSS (no script needed to cause harm).

- **Classic use**: inject a `<form>` that POSTs credentials to your domain — phishing hosted on the *real* site.
- **Coinbase case**: injected `<a href>`/form in comment fields ($500-class).
- **Test**: submit `<h1>`, `<a href=//evil>`, `<form action=//evil>` → rendered or escaped?

## HTTP Parameter Pollution (HPP)

App forwards user input into a *second* HTTP request (server-side or client-side) without validation — duplicate-param games.

**Server-side model**: `transferMoney.php?amount=1000&fromAccount=12345` forwards to backend with fixed `toAccount=9876`. Submit `&toAccount=99999` → backend sees `toAccount=9876&...&toAccount=99999` → which wins depends on stack.

**Duplicate-param precedence** (critical to know):

| Stack | Wins |
|-------|------|
| PHP/Apache | **last** occurrence |
| Apache Tomcat | **first** occurrence |
| ASP/IIS | **all** (concatenated) |

**Client-side model**: param reflected into a generated link. `?par=123%26action=edit` → after `htmlspecialchars`, `%26` decodes to `&` → link gains `&action=edit` → extra param smuggled through an "escaped" field.

**Cases:**
- **HackerOne social share buttons** ($500): `?&u=https://vk.com/durov` appended — Facebook sharer took the **last** `u=`; Twitter default `text=` similarly overridable. *Takeaway: watch where submitted content flows to other services (share links, webhooks).*
- **Twitter unsubscribe** ($700): `uid` alone → error; **second** `uid=` param → unsubscribed *another* user. *Takeaway: persistence — the first failure isn't the answer; duplicate the param.*
- **Twitter Web Intents**: two `screen_name` params → UI shows first, form submits second → follow-the-wrong-user. *Takeaway: an HPP hit suggests a systemic issue — sweep every intent/endpoint.*

## CRLF Injection

`%0d%0a` (`\r\n`) terminates lines in HTTP — inject it where input reaches headers → response splitting, arbitrary headers, even smuggling.

- **Test**: `%0d%0a` in any param whose value lands in a response header (cookies are the prime spot — `Set-Cookie` embeds user input by design).
- **Twitter case** ($3,500): `0x0a` was filtered — bypassed by **UTF-8 encoding the newline**: `%E5%E98%8A` → decoded server-side back to `0A`. Then chained: `redirect_after_login=https://twitter.com:21/%E5%98%8A…content-type:text/html…svg/onload=alert(innerHTML)` → header injection delivering XSS → session theft. *Takeaway: submit re-encoded/double-encoded values when filters block the plain form; knowledge of old encoding bugs is ammunition.*
- **v.shopify.com** ($500): `%0d%0aContent-Length:%200%0d%0a%0d%0aHTTP/1.1%20200%20OK%0d%0a…<html>deface</html>` in the `last_shop` cookie param → **full second response** → XSS-grade impact. *Takeaway: input that becomes a cookie value is a CRLF candidate by default.*

## The detection habit

- Where does my input go? Page body → HTMLi/XSS. Another request → HPP. A header/cookie → CRLF.
- Filter blocking a char → re-encode (`%250d` → `%0d`), UTF-8 overlong forms, null bytes.
- One finding of a class → assume systemic → enumerate every similar feature (Twitter lesson).
