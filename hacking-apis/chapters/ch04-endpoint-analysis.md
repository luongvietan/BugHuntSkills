# Ch7: Endpoint Analysis

Source: Chapter 7. Turn discovered APIs into a working request collection, use them as intended, and score early wins: info disclosures, misconfigurations, excessive data exposure, business-logic flaws.

## Step 1 — Get request information

**Documentation** — try `/docs`, `/api/docs`, `docs.`, `dev.`, `developer.`, `/developers/documentation`. If locked down: register an account and look again, Google-dork it, check Wayback for retracted docs, fuzz doc paths (`subdomains_list`, `dir_list` from the Hacking-APIs repo). Admin docs are often public by self-service design — they're the BFLA target list.

**Docs are never complete.** Test methods, endpoints, params *not* in the docs. Convention cheat sheet: `:id`/`{id}` = path var, `[name]` = optional, `a || b` = alternatives, `<x>` = typed string.

**Specification import** — find OpenAPI/Swagger (`"swagger":"2.0"`), RAML, or a Postman collection → Postman Import → Link → instant collection. Check collection Variables tab; add `{{hapi_token}}` etc.

**Reverse engineering** — no docs: (a) build requests manually into a collection with a `{{baseURL}}` variable (lets you re-version hundreds of requests at once — swap `v1`→`v2`→`v3` by editing one variable); (b) proxy browser traffic through Postman (Capture requests, port 5555 + FoxyProxy), use *every* feature — register, login, password reset, all links, profile update, forums, shop — then prune non-API requests and group by endpoint.

**Auth requirements** — replicate the documented flow (POST creds → token, OAuth, or out-of-band). Save the working auth request; store token as variable. Then feed the token to Kiterunner (`-H 'x-access-token: …'`) to re-scan for endpoints that only appear authenticated.

## Step 2 — Use it as intended (build the baseline)

Send documented requests with required headers/params/auth until 200s. While doing so ask:

- What actions exist? Can I interact with other accounts? What resources exist?
- How are new resources identified (sequential int? GUID? email? composite)?
- Can I upload/edit files?
- What do admin actions look like (docs)? Test them unauthenticated → low-priv → admin.

Privileged actions in docs = top priority. If they're properly gated, that's your *goal* for later chapters (get an admin token).

## Step 3 — Analyze responses for early wins

Baseline = normal status code + size + timing + body shape. Deviations are findings:

- **Information disclosures**: software versions, usernames, emails, phone numbers, account numbers, partner names in responses; `X-Powered-By`-style headers; status-code oracles (404 vs 401 → enumerate valid users/accounts/phones — feeds password spraying, phishing, BOLA combos).
- **Verbose errors**: different messages for "user doesn't exist" vs "wrong password" → username enum.
- **Transit crypto gaps**: any sensitive data over HTTP? Can a consumer force HTTP and leak tokens (Wireshark)? Even with HTTPS server-side, check the client can't initiate plaintext.
- **Debug/misconfig pages**: Django/Express debug output discloses framework + full endpoint map.
- **Excessive data exposure**: response carries fields the UI never displays — MFA status, activation state, `is_admin`, other users' data. Scale test with Collection Runner. Extra data is only a vuln if it's *attack-useful* — but "username + last-login timestamp" is already recon fuel.
- **Business logic**: docs explain intended rules; look for where the rules have no enforcement (wildcards, negative amounts, skipped steps).

Track which disclosures you used — the report should show how a "minor" info leak enabled the real exploit.
