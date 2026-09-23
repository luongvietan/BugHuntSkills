# Ch 4 — Exposed Databases

> **Exposure ceiling:** connection success + a database/collection listing proves the misconfig — that is the entire PoC. Never read rows, never sample "just one record": if the listing shows personal-data names (users, customers, patients), report the listing itself.

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
GET /_cat/indices?v          → list all indexes ("databases") ← the PoC ceiling
GET /_stats/?pretty=1        → service details
GET /INDEX/_mapping?pretty=1 → list field names (columns)
```

The listing proves the misconfig — an index named `users`/`customers` plus
field names like `email`, `password`, `token` is the report. Search
queries (`_search?q=…`) return document *contents* = reading real user
data; they are not part of a bounty PoC.

## MongoDB recipe (port 27017)

No auth by default. `mongo <ip>` connects → `show dbs` listing is the
proof ceiling. `unauthorized` error = auth enabled (move on).

## Firebase recipe

Any `*.firebaseio.com` reference → `https://<name>.firebaseio.com/.json`.
An unauthenticated 200 with data confirms exposure — capture the response
*shape* (top-level keys, redacted), not the export.

## Play

- These are misconfigurations, not exploits — impact is immediate (PII/creds → report fast, never over-extract).
- A found credential is reported, not exercised — login chains are escalation narrative, never a live step (`hacking-the-cloud` ch03 rule).
