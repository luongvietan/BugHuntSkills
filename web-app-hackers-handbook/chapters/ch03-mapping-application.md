# Ch4: Mapping the Application

> **Edition note (2011):** the mapping *discipline* is durable; the surface has
> grown — add JS-bundle endpoint extraction, API/GraphQL discovery, SPA
> routing, and subdomain/cloud asset enumeration to this chapter's checklist.
> Modern procedures: `web-security-academy` (API/graphql) and
> `recon-pipeline` (asset discovery).

Source: Chapter 4. Mapping builds the attack-surface inventory everything else depends on: you cannot test a parameter you haven't found. Output = a catalogue of content, functionality, entry points, and technology guesses.

## Enumeration — content & functionality

- **Spider + manual walkthrough**: automate crawling, then *manually* exercise every function at every privilege level — spiders miss JS-driven, multistage, and authenticated flows. Note every distinct function and its parameters.
- **Monitor passive/published sources**: Wayback Machine, Google (`site:`, `cache:`), search caches — historical content, removed-but-still-deployed pages, backup files.
- **Discover hidden content**: brute-force directories/files (`/admin`, `/test`, `~user`, backup extensions `.bak .old .swp .tmp`, config files). Default content ships secrets. Don't trust robots.txt/`Disallow` as hiding — it's a treasure map.
- **Hidden parameters**: the server honors params never sent by the UI — `debug`, `admin`, `sort`, `user_id`, `role`. Infer names from observed conventions, JS, error messages, and defaults. Try adding them.
- **Entry points beyond params**: every HTTP header the app may process (Referer, User-Agent, X-Forwarded-*), cookies, out-of-band channels (email→app, upload→processing), and any per-user shared state.

## Fingerprinting

- **Platform**: banner/Server header, error messages, file extensions (`.aspx`, `.jsp`, `.php`, `.cfm`, `.pl`), framework cookies (`ASP.NET_SessionId`, `JSESSIONID`, `PHPSESSID`), URL conventions, directory names, cookie name prefixes.
- **Extrapolate**: each technology guess suggests default content paths, known framework bugs, and language-typical vuln classes (PHP → LFI/RCE; Java → deserialization; .NET → viewstate).
- **Client-side tech**: JS libraries and versions (old = old bugs), applets, Flash/Silverlight objects, ActiveX — each is an attack surface with its own chapter (ch04 client controls, ch12 user attacks).

## Map the defenses while mapping

- Record access-control boundaries (which URLs require which role), session mechanism (cookie names, token format), where input validation appears to live, WAF presence (provoke a 403 with a blatant probe), rate limits.

## Checklist

- [ ] Full walkthrough at each privilege level; every request logged.
- [ ] Content enumeration: spider results + brute-forced dirs + archive/search results merged.
- [ ] Every entry point catalogued: params (query, body, cookie, path), headers, uploads, out-of-band.
- [ ] Hidden-parameter candidates tried; multistage flows documented stage-by-stage.
- [ ] Technology fingerprint → list of suggested default paths and likely bug classes.
- [ ] Defense inventory: auth boundaries, session tokens, filters, WAF, rate limits.
