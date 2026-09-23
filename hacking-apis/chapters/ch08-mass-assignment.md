# Ch11: Exploiting Mass Assignment

Source: Chapter 11. The API binds client-supplied JSON fields to server-side object variables without filtering → add variables the UI never offers.

## Where to look

Requests that accept and store client input: **account registration** (most common — try `"admin":true` at signup), password reset, profile/account/group/company update, order/checkout, any `PUT`/`PATCH`/`POST` writing an object.

Registration example:

```json
POST /api/v1/register
{"username":"hAPI_hacker","email":"hapi@hacker.com","admin":true,"password":"Password1!"}
```

Org-hijack variant: add `"org":"CompanyA"` (fuzz the org value if it's an ID — BOLA-style brute force).

## Finding the right variable names

Inconsistency is the challenge — hunt the provider's naming conventions:

1. **Docs/specs**: compare low-priv vs admin request examples. `POST /api/admin/create/user` shows `"admin": true` → replay that same field on the public `/api/create/user` with a low-priv token.
2. **Bonus params in captured requests**: `{"uam":1,"mfa":true,"account":101}` — flip values (`uam:0`, `mfa:false`, `account:0-101`) and watch responses. Params seen on one endpoint may be honored on another.
3. **Blind spray**: send many candidate variables in one request — the API often ignores unknowns and binds the right one:

```json
{"username":"hAPI_hacker","email":"hapi@hacker.com",
 "admin":true,"admin":1,"isadmin":true,"role":"admin",
 "role":"administrator","user_priv":"admin","password":"Password1!"}
```

4. **Arjun** (param discovery): `arjun --headers "Content-Type: application/json" -u http://target/api/register -m JSON --include='{$arjun$}'` → heuristic + wordlist → valid params (`user`, `pass`, `admin`). `--stable` slows for rate limits.
5. **Many-request spray** (if 400/401/413 rejects big bodies): Intruder payload position on a single extra field, wordlist of candidate variable names.

## Chaining: BFLA + mass assignment = account takeover

The book's combo: BFLA lets UserA edit *basic* fields on Brock's profile (username/address). Add mass-assigned fields to that same PUT:

```json
PUT /api/v1/account/update  (UserA token, targeting Brock)
{"username":"Brock","email":"ash@email.com","mfa":false}
```

Email now yours → trigger password reset → temp password arrives in *your* inbox → MFA disabled → full takeover. Mass assignment turns a low-impact write into critical compromise.

## Notes

- If mass assignment works, expect more vulns nearby — it's "the tip of the iceberg."
- Watch GET responses for sensitive variables (`"credit"` on product objects) then hunt a write method on the same endpoint (`§GET§`→POST/PUT fuzz; a 400 response listing required fields literally hands you the schema).
