# Ch24: API Hacking

> **Current taxonomy:** route to `owasp-api-security-top-10` for the 2023 risk list (BOLA/BOPLA/business-flows/SSRF/unsafe-consumption) and `hacking-apis` for the test method — this chapter is the intro-level version.

Source: Chapter 24. APIs = the same vuln classes behind a thinner UI and often weaker controls. Mobile apps and SPAs are API-clients — hack the API, not just the page. Deeper treatment in companion skill `hacking-apis`.

## API shapes & recon

- **REST**: predictable `RESOURCE/ACTION` structure (`/1.1/statuses/show.json?id=…`), HTTP-method semantics (GET/POST/PUT/DELETE), JSON bodies.
- **SOAP**: XML envelopes, older/IoT apps; **WSDL** = built-in API map — find it via `?wsdl`, `.wsdl`, `inurl:wsdl`.
- **GraphQL**: single endpoint (`/graphql`, `/gql`), `query` = read, `mutation` = write; **introspection** (`{__schema{types{name}}}`, `{__type(name:"X"){fields{name}}}`) = self-documenting schema → instant endpoint/field map. Disabled introspection → **Clairvoyance** (brute-force schema via error suggestions) or guess from queries.

## Enumeration (the hard part is knowing what exists)

1. Public docs: `company API`, `developer docs` searches.
2. Internal docs left open: `company inurl:swagger` (Swagger/OpenAPI JSON = full endpoint+param list).
3. Walk all app workflows with proxy on — capture API calls the docs don't list (private endpoints).
4. Deduce siblings: `/posts/ID/read` + `/posts/ID/delete` → try `/posts/ID/edit`. REST is guessable.
5. JS source + GitHub for hidden endpoints/params.
6. **Old API versions**: `/api/v2/user_emails/X` exists → try `/api/v1/user_emails/X` — old versions keep old bugs.
7. Error-forcing: wrong types, malformed JSON → verbose errors leak internals.
8. Wordlist fuzzing for endpoints (API-specific lists — ch22).

Auth model mapping (do this before attacking): which endpoints need tokens? how are tokens minted? can you mint without login? do tokens die on password reset?

## The three high-yield test classes

1. **Broken access control & info leaks** (the most common API bugs):
   - Token validation: not validated? predictable (low entropy → Sequencer)? never expires?
   - Object-level: swap IDs in path/body (IDOR logic — ch07); GraphQL mutations from a lower-priv account.
   - Function-level: same action, two endpoints (`POST /posts/ID/delete` vs `DELETE /posts/ID`) — are *both* protected?
   - Response over-sharing: API returns fields the page never renders (the book's private-token-in-profile leak → impersonation). Always read raw responses.
   - Method/params tricks: switch GET→PUT→PATCH→DELETE; add `admin=1`, `role=`, `user_id=` params.
2. **Rate limiting gaps**: no limit on auth/OTP/search/data endpoints → brute-force & harvesting. Test with Burp Intruder/curl 100-200 requests at each privilege level — **get written permission + throttle** (easy to DoS a small target; some limits exceed your tools anyway).
3. **Technical bugs through the API**: API input = another injection surface — SQLi/deserialization/XXE (XML bodies!)/SSRF (URL-fetching endpoints)/race conditions/file-upload RCE/path traversal via filepath params/reflected XSS via URL params echoed in JSON→rendered pages. Same payloads, API packaging.

Tools: Postman (collection management), GraphQL Playground, ZAP GraphQL add-on, Burp + Wfuzz.

## Checklist

Docs+WSDL+introspection harvested → endpoints enumerated incl. hidden/old versions → token model mapped → BOLA/function/method tests → response-diff for leaks → rate-limit check (permitted) → standard vuln classes through API inputs.
