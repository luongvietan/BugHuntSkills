# Ch22: Conducting Code Reviews

Source: Chapter 22. White-box advantage: see the logic, not just the behavior. Access comes from `.git` leaks, open-source components, mobile APKs (ch20), client-side JS — always present even in "black-box" work.

## Approach: breadth first, then depth

**Phase 1 — grep sweep** (minutes, high yield):

```
grep -rniE "password|passwd|secret|api_?key|token|private" .
grep -rniE "https?://" .                     # endpoints, hardcoded hosts
grep -rniE "TODO|FIXME|HACK|XXX|debug" .     # unfinished validation
grep -rniE "eval|exec|system|passthru|shell_exec|os\.system|subprocess|deserialize|unserialize|pickle|ObjectInputStream|include|require" .
```

Harvest: hardcoded creds, hidden endpoints, admin/debug functions, config files, interesting comments (they point at known-broken code).

**Phase 2 — sensitive-function review**: find the code behind authentication, password reset, state changes, sensitive reads — then audit each path.

**Phase 3 — source→sink tracing**: pick every user-input source (HTTP params, headers, path segments, cookies, DB values that originated from users, file reads, uploads) and follow it to sinks (SQL, shell, template, redirect, response, deserialization). Any path that crosses a trust boundary unsanitized = bug.

## Patterns the book's examples expose

- **SQLi by concat**: `query = "SELECT * FROM users WHERE user='" + username + "'"` — any string-built query is suspect; check *every* param that reaches it.
- **Open redirect via substring**: validator does `if "example.com" in redirect_url` → `attacker.com/?q=example.com` passes. Whitelist checks must be on the *host*, not the string.
- **CSRF by optional validation**: `if "csrf_token" in request: validate()` → omit the param, skip the check entirely. Also: Referer check that only asserts *presence*.
- **GET-based state change**: `GET /change_password?new=…` — no CSRF defense possible + secret in URL/logs.
- **Dangerous defaults**: debug mode flags shipped on, permissive CORS helpers, commented-out auth checks, `except: pass` swallowing validation failures.
- **Comments as a map**: `// TODO: add auth check`, `// temporary bypass`, `// remove before prod` — treat every one as a lead.

## SAST as force multiplier

After manual fundamentals: Semgrep (custom rules fast), CodeQL (deep queries), language linters/bandit/brakeman/sonarqube. Use them to *find candidate sinks*, then hand-verify exploitability — scanner hits without reachability analysis are noise.

## Checklist

Sources enumerated → sinks enumerated → grep sweep done → sensitive paths hand-reviewed → every trust-boundary crossing traced → comment/TODO leads exhausted → findings verified dynamically when possible.
