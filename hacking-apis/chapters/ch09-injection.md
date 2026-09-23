# Ch12: API Injection

Source: Chapter 12. Input reaches an interpreter (DB, OS, browser, JSON parser) unsanitized. API injection is the same classic bug classes with a new delivery channel — often better-defended, so detection is the hard part.

## Injection points

Fuzz: API keys, tokens, headers, URL query strings, POST/PUT body params. Detect via verbose errors, DB errors, anomalous status codes, timing. **Found one? Test every similar endpoint** — `/file/upload` vulnerable ⇒ `/image/upload`, `/account/upload` probably are too.

## XSS via API

API → webapp rendering path: profile updates, "like"/social fields, product data, forum/comment posts. Submit payloads through the *API request* (devs often guard the web form but not the API):

```json
POST /api/profile/update {"city":"<script>alert('xas')</script>"}
```

Load the rendered page to confirm. Payloads: `<script>alert(1)</script>`, `<%00script>…`, `SCRIPT>alert(1)///SCRIPT>`. Lists: PayloadBox xss-payload-list (~2700), Wfuzz wordlists, NetSec.expert. Tip: try `Content-Type: text/html` to coax HTML parsing.

## XAS (Cross-API Scripting)

Script enters via a *third-party* API the target consumes (e.g., LinkedIn feed → blog sidebar), or via the provider's own API writing to its webapp. Same payloads, extra preconditions: downstream render + poor sanitization at some hop. Defenders fix XSS but overlook XAS.

## SQL injection

- **Request the unexpected** (per docs types): string where number expected, huge number where small expected, non-boolean for boolean — verbose DB errors fingerprint SQL.
- **Metacharacters**: `'`, `''`, `;%00`, `--`, `-- -`, `""`, `;`, `' OR '1`, `' OR 1 -- -`, `" OR ""="`, `" OR 1=1-- -`, `OR 1=1`, `' OR ''='`. Classic auth bypass: `' OR 1=1-- -` comments out the password check.
- **SQLmap** on a saved request: `sqlmap -r req.txt -p param` (omit `-p` = test all; `CTRL-C`+`n` skips slow params). Then `--dump-all`, or `--dump -T users -C password -D helpdesk`, or `--os-shell` / `--os-pwn`.

## NoSQL injection — *higher-yield than SQLi on modern APIs*

NoSQL is common in APIs and less understood by defenders. MongoDB-style payloads:

`$gt`, `{"$gt":""}`, `{"$gt":-1}`, `$ne`, `{"$ne":""}`, `{"$ne":-1}`, `$nin`, `{"$nin":1}`, `{"$nin":[1]}`, `{"$where":"sleep(1000)"}`, `||'1'=='1`, `'||'1'=='1';//`, `'"\\;{}`, `'/{}:`, `'"\\/$[].>`

Tell: `{"$gt":""}` or `{"$ne":""}` as a **password** → auth bypass (condition always true). Time-based: `{"$where":"sleep(1000)"}` → 10s delay confirms. A `SyntaxError`/400 on `'"\\;{}` = input-parsing weakness worth pursuing.

## OS command injection

Separators: `|`, `||`, `&`, `&&`, `'`, `"`, `;`, `'"`. Fuzz with two positions — separator × command — cluster bomb or Wfuzz two `-z file` lists.

Commands: *nix `whoami`, `id`, `uname -a`, `pwd`, `ls`, `ifconfig`, `cat /etc/passwd`; Windows `whoami`, `ver`, `dir`, `ipconfig`, `echo %CD%`. Nmap OS fingerprinting first → right command list.

**Payload-position trap (from Lab #9):** intruder positions inside existing JSON quotes → `{"coupon_code":"§TEST§"}` produces `{"coupon_code":"{"$nin":[1]}"}` (broken JSON, 422). Move position to *include* the quotes: `{"coupon_code":§"TEST!"§}` → clean nested-object injection → 200 + valid coupon. Also uncheck Intruder's URL-encode-this-characters box when encoding breaks the payload.
