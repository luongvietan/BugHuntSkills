# Ch14: Attacking GraphQL

Source: Chapter 14 (lab: DVGA). GraphQL ≠ REST: **one endpoint, POST only**; body carries `query` (read), `mutation` (write), `subscription` (realtime). Queries declare object type + args + wanted fields — think "SQL with extra steps". Most endpoints return **200 with errors in the body**, so status-code anomaly detection doesn't apply — diff response bodies/lengths instead.

> **Bounded-probing rules (2026 refresh):** introspection queries and
> nested/circular query tests are *expensive by design* — a deep nesting bomb
> is a DoS probe. Run the single standard introspection query once; map the
> schema from that result; don't loop it. Query-cost/depth tests (does the
> server cap depth or complexity?) are demonstrated with ONE escalating
> sequence, not a flood. Batching/alias attacks: a small aliased batch proves
> the missing cost limit — never a 1000-alias packet. Suggestions/error
> leakage stays the low-noise discovery path.

## Finding the endpoint & IDE

- Dir-brute with GraphQL wordlists (`kr brute target -w seclists/…/graphql.txt`): `/graphql`, `/v1/graphql`, `/api/graphql`, `/graph`, `/graphiql`, `/console`, `/query`, `/graphql/console`, `/altair`, `/playground` + `/v2 /v3 /test /internal /mobile /legacy` variants.
- IDEs (GraphiQL, Playground, Altair) often ship with the app — the Documentation Explorer is auto-generated API docs.
- **Cookie tamper to unlock the IDE (DVGA trick):** response sets `Set-Cookie: env=graphiql:disable` (base64 `env=Z3JhcGh…`). Decode → flip `disable`→`enable` → re-encode → write it in DevTools Storage → IDE + Docs Explorer unlocked.

## Reverse engineering

- Proxy app traffic through Postman; delete non-`/graphql` requests; **rename by reading the body** — all requests look identical from the URL.
- **Introspection = jackpot.** GraphiQL fires `IntrospectionQuery` to populate its docs; replay it yourself if no IDE:

```graphql
query IntrospectionQuery { __schema { queryType{name} mutationType{name}
  subscriptionType{name} types { ...FullType } directives { name } } }
```

Full schema = every type/field/arg = your Postman collection. If disabled, fall back to body-reading + InQL guessing.
- **InQL Burp extension** (Jython): InQL Scanner → target URL → auto-enumerates queries/mutations → send to Repeater.

## Attacking

- **BOLA via object args**: `paste.query(pId:§)` — brute-force sequential pIds in Intruder → private pastes (`"public":false`) returned. Any ID-ish argument is a BOLA surface.
- **Fuzz the mutation variables**: mutations take `"variables":{"host":"…","path":"/","port":80}` — put payload positions inside variable values; separator × command cluster-bomb (metachars + `whoami`, `uname -a`, `cat /etc/passwd`). DVGA: `path` injects OS commands as root.
- **Baseline bodies, not codes:** errors and successes both come back 200 — sort by response length and diff bodies for anomalies; errors still leak info.
- **Excessive fields:** request extra fields per object (`ipAddr`, `ownerId`, `userAgent`…) — the schema tells you what exists; ask for what the UI never shows.
- Standard techniques port over: injection payloads go in args/variables; auth bypass = call queries/mutations unauthenticated; BFLA = call admin mutations with a low-priv token.
