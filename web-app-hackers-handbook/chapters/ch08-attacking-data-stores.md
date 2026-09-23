# Ch9: Attacking Data Stores

Source: Chapter 9. Injection = the app interprets user input as *code* rather than *data* at some interpreter boundary: SQL, OS shell, LDAP, XPath, NoSQL, XML. Signature mechanics: the input modifies the query's **syntax**, so the same payload family works wherever the boundary exists.

## SQL injection — mechanism

- Query structure is string-concatenated with user input → `'` in a string context or bare value in numeric context changes query logic. Auth bypass (`admin'--`), data extraction, filter evasion, write/exec escalation.
- **Detection ladder**: submit `'`, `"`, `` ` ``, `\` → error messages, 500s, or *subtle deltas* (missing row, different length, timing). Then **tautology vs contraction**: `OR 1=1` (all rows) vs `AND 1=2` (none) proves logic injection without touching real data.
- **Confirm blind**: `; waitfor delay '0:0:5'--` (MSSQL), `SLEEP(5)` (MySQL), `pg_sleep`, `dbms_pipe.receive_message` (Oracle) — timing is an oracle when nothing returns.
- **UNION extraction**: match column count (`ORDER BY n` increments or `UNION SELECT NULL,NULL,…` until error disappears) → find a *string-typed* column (`'a'` probe each position) → `UNION SELECT NULL,version(),NULL` → schema via `information_schema` (MySQL/PG) / `sqlite_master` / `all_tables` (Oracle) / `sysobjects` (MSSQL).

## Blind & inference techniques

- **Boolean-blind**: condition in WHERE (`AND (SELECT SUBSTRING(password,1,1))='a'`), content delta = oracle. Automate char-by-char extraction.
- **Time-blind**: `CASE WHEN condition THEN delay ELSE 0` — measure, repeat to average jitter.
- **OOB channels**: `UTL_HTTP`/`UTL_INADDR` (Oracle DNS/HTTP), `xp_dirtree`→DNS (MSSQL), `LOAD_FILE`→`INTO OUTFILE` (MySQL) — exfil via DNS/HTTP to your listener when responses are unreachable.
- **Second-order SQLi**: payload stored safely (escaped) then concatenated into a *different* query later (username → admin reporting). Test stored fields; injection fires where you can't see — use time/OOB confirmation.

## Bypassing defenses

- Filters strip spaces → `/**/` comments, tabs/newlines (`%0a`), `UNION%0aSELECT`.
- Keyword filters → case mix, nested keywords surviving single-pass strip (`SELSELECTECT`), inline comments `/*!UNION*/` (MySQL executes it), `un'union'ion`.
- Quote escaping/doubling → inject into **numeric context** (no quotes needed); truncation-after-escape (ch10 logic ex.9: pad to limit so doubled quote truncates to a lone `'`); GBK/multibyte `%bf%27` where `\` becomes part of a wide char.
- WAF in front → same catalog + NULL bytes (`%00` truncates the WAF's string in native code).

## Escalation through the DB

- File read/write: `LOAD_FILE`, `INTO OUTFILE`, `xp_cmdshell` (MSSQL → OS RCE), bulk insert → webroot write.
- DB→OS: UDFs, `xp_*` procs, CLR/Java stored procs, external tables.
- Chained: SQLi → creds → login; SQLi in an *insert* → stored payload reaching an admin (second-order stored XSS, ch11).

## Beyond SQL — same injection principle, different interpreter

- **XPath injection**: login `user' or '1'='1` on XML-stored creds; `']|//*|/*['` to dump whole docs; blind extraction char-by-char.
- **LDAP injection**: `*)(uid=*))(|(uid=*` filter manipulation → auth bypass / directory dump; `*` wildcards enumerate attributes.
- **NoSQL (MongoDB-style)**: operators in params — `user[$ne]=x&pass[$ne]=x` → auth bypass; `$where` JS evaluation; JSON body injection `{"$gt":""}`.

## Checklist

- [ ] Every param: metachar probes → error/delta → boolean confirm → time confirm.
- [ ] Numeric contexts tested without quotes; string contexts with quote escape variants.
- [ ] UNION count + column-type mapping → `version()`/own-row PoC (minimal).
- [ ] Stored inputs retested for second-order execution.
- [ ] Non-SQL interpreters mapped: XML, LDAP, search, NoSQL — same probes, their syntax.
- [ ] OOB channel attempted before declaring "not exploitable."
