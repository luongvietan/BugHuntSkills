# Ch 4 — Exposed Databases

Databases are frequently internet-exposed **without authentication by default**. Zero-skill, maximum-impact: connect → dump.

## Indicator table

| DB | Indicator | Test |
|----|-----------|------|
| Firebase | `*.firebaseio.com` URL in requests | Append `/.json` → dumps whole DB if unauthenticated |
| Elasticsearch | Port `9200` open | GET `/` returns version JSON → enumerate |
| MongoDB | Port `27017` open | `mongo <ip>` → run commands; "unauthorized" = secured |
| CouchDB | Ports `5984`/`6984` | HTTP API, default unauth |
| Cassandra | Ports `9042`/`9160` | cqlsh connect |

## Elasticsearch recipe (port 9200)

Unauthenticated by default; find via Shodan or port scans.

```
GET /                        → confirms ES + version
GET /_cat/indices?v          → list all indexes ("databases")
GET /_stats/?pretty=1        → service details
GET /_all/_search?q=email    → full-text search across all indexes
GET /INDEX/_mapping?pretty=1 → list field names (columns)
GET /_all/_search?q=_exists:email&pretty=1 → docs having an "email" field
```

**Search keywords**: `username`, `email`, `password`, `token`, `secret`, `key`.
Replace `_all` with a specific index name to scope queries.

## MongoDB recipe (port 27017)

No auth by default. `mongo <ip>` → issue any command. `unauthorized` error = auth enabled (move on); arbitrary command execution = full dump.

## Firebase recipe

Any `*.firebaseio.com` reference → try `https://<name>.firebaseio.com/.json`. If auth wasn't enabled, the entire realtime DB exports as JSON — often including plaintext passwords.

## Play

- These are misconfigurations, not exploits — impact is immediate (PII/creds → report fast, don't over-extract).
- Combine: dumped creds → login to app → deeper access.
