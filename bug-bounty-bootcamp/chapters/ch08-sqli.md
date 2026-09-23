# Ch11: SQL Injection

Source: Chapter 11. SQLi = user input reaches a SQL query unsanitized, letting the attacker alter the query's logic. Classic PoC chain: `'` breaks the query → error reveals structure → confirm with `' OR '1'='1` (tautology) or `AND '1'='2` (contraction) → extract via UNION/time.

## Hunting

- **Entry points**: any param in URL/body/cookie/header — IDs, search, sort fields (`ORDER BY` injection is common and often missed), filters.
- **Detection ladder**: `'`, `"`, `` ` ``, `\`, `%27`, `%22` → watch for 500s, SQL error strings (`you have an error in your sql syntax`, `unclosed quotation`, `ORA-`), content-length deltas, missing rows.
- **Confirm safely**: `' OR 1=1-- ` returns all rows (vs `' OR 1=2-- ` returns none) = boolean-blind confirmation without dumping data. `'` + `SLEEP(5)` / `pg_sleep` / `WAITFOR DELAY` = time-blind confirmation.
- **UNION extraction**: match column count with `UNION SELECT NULL,NULL,…` until no error, then `UNION SELECT NULL,version(),NULL…` → proves data access without touching real rows.
- **DB fingerprinting** via error text + function names: `@@version`/`sqlite_version()`/`version()`.

## Bypassing protections

- WAF strips spaces → `/**/` comments, tab/newline, `UNION%0aSELECT`.
- Keyword filters → `UnIoN SeLeCt`, `un'union'ion`, `selselectect` (nested keywords survive single-pass strip), `/*!UNION*/` MySQL comment-trick.
- Quotes escaped → try numeric context (`id=1 AND 1=1` — no quotes needed), or GBK/wide-char trick `%bf%27` eats the backslash.
- Second-order SQLi: payload stored safely, executed later (username → admin panel query) — test stored fields with delayed probes.
- `sqlmap` evasion: `--tamper=space2comment,between,randomcase` + `--delay` to stay under rate limits.

## Escalation & impact

Read path: DB version → schema (`information_schema.tables/columns`) → table names → rows. The book's caution: prove access with `version()`, a hash, or your *own* row — dumping users' data violates the program. Write/exec paths: `INTO OUTFILE`/stacked queries → file write → webshell; `xp_cmdshell` on MSSQL → RCE. Auth bypass `admin'-- ` is the fastest demonstration of real impact.

## Automation & first bug

`sqlmap -u "URL?p=1" --batch --level=3 --risk=2` — start low (level/risk) on in-scope GET params; `-r request.txt` for full captured requests; `--technique=T` for time-based. Manual-first rule: sqlmap noise can hit prod hard — throttle, whitelist scope, never `--dump-all`.

**First-SQLi checklist**: fuzz params with quote chars → error/delta → boolean confirm → UNION count → `version()` PoC → optional sqlmap confirmation → stop, report.
