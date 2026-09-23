# Ch 9 — API Types: REST, RPC, SOAP, GraphQL

Modern apps split: JS frontend (React/Angular) ↔ backend API. **Step one is always: which API type?** — detection hints below. Then all the normal OWASP bugs apply on top (SQLi, XSS, IDOR…).

## REST

- **Tells**: JSON request/response bodies; verbs beyond GET/POST.
- Methods: `GET` read · `POST` create (often update too) · `PUT`/`PATCH` update · `DELETE` delete.
- A `PUT` with JSON `{"param1":"value1"}` = updating a resource — instantly tells you the object's writable fields (mass-assignment / IDOR surface).

## RPC (XMLRPC / JSONRPC)

- **Tells**: only GET/POST; one HTTP request ↔ one function call; filenames like `xmlrpc.php`; body contains `methodCall`/`methodName` (XML) or `{"method":…}` (JSONRPC).
- `<methodName>system.listMethods</methodName>` — enumerate exposed functions; a goldmine for hidden admin methods.

## SOAP

- **Tells**: XML body wrapped in `<soapenv:Envelope>` (Header optional — auth/metadata; Body — the call).
- Body = method + typed args: `<web:GetCitiesByCountry><web:CountryName>x</…>`.
- Docs via **WSDL** (see ch10). XXE surface everywhere — it's XML.

## GraphQL

Single endpoint replaces many REST routes; one request queries whatever fields you ask for. **No auth by default** — developers must add it; many forget.

**Discovery** (add to dir-bruteforce): `/graphql`, `/graphiql`, `/graphql.php`, `/graphql/console`.

**Introspection** — dump the whole schema:

```
/graphql?query={__schema{types{name,fields{name}}}}
```

→ reveals types + fields (e.g. `User{username,password}`). Ignore `__`-prefixed types (introspection internals). Then query directly:

```
/graphql?query={User{username,password}}
```

→ unauthenticated credential dump in the author's example. GraphQL also carries IDOR inside queries — object fields often trust whatever id you pass.

## Play

1. Identify type from request shape.
2. Find docs (ch10) → enumerate operations.
3. Test auth on every operation + standard OWASP classes per parameter.
