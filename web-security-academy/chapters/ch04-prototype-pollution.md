# Ch04: Prototype Pollution — Client & Server

Source: `/web-security/prototype-pollution`. **GAP CLASS — no book covers this.** A JavaScript bug where an attacker adds properties to `Object.prototype` (or other built-in prototypes); every object that doesn't own the property then inherits the attacker's value.

> **Lab vs live (bounty-safe):** server-side PP gadgets can alter app behavior
> for *everyone* (status-code override, maintenance mode, shell injection).
> Confirm pollution with a canary property visible only to your own
> session/request; a gadget that flips shared state = stop at proof-of-
> pollution and report — do not chain to RCE on a live program.

## Mechanism

Every JS object inherits from a prototype chain ending at `Object.prototype`. A recursive merge/assign that copies attacker-controlled keys *without sanitizing* lets `__proto__` (or `constructor.prototype`) slip through: the merge writes to the prototype object, not the target. JSON input is a prime vector because `JSON.parse('{"__proto__":{...}}')` produces a real `__proto__` key (unlike an object literal, where it's the setter).

Exploitation needs three parts — missing any one = trivia, not a bug:

1. **Source** — input that poisons prototypes: URL query/fragment (`?__proto__[x]=y`), JSON bodies, web messages (`postMessage` data merged into objects).
2. **Sink** — code path where the polluted property does harm: script URL assignment, `eval`/`innerHTML`, template config, `child_process` options.
3. **Gadget** — a property the app reads without checking, passing your value into the sink (e.g. a `transport_url`-style config property, `shell`, `input`, or a property consumed by a vulnerable library).

## Client-side prototype pollution

- **Manual source hunting**: in DevTools, run `Object.prototype` after injecting `?__proto__[test]=test` via URL/hash/JSON — a polluted property on `{}` confirms the source. Try `__proto__`, `constructor.prototype`, `__proto__.x`, and encoded/bracket forms.
- **DOM Invader** (Burp's browser) has a dedicated prototype-pollution mode that finds sources and scans for reachable gadgets — the intended tool.
- **Via the constructor**: `?__proto__` blocked? Try `?constructor[prototype][x]=y` or `constructor.prototype.x` — same object reached through the prototype chain.
- **Flawed key sanitization bypass**: filters strip literal `__proto__` but miss the other routes to the same object — `__proto__` inside *nested* objects (filter checks only top-level keys), unicode/encoding variants that decode after the check, and `constructor.prototype` / `constructor[prototype]` chains that reach `Object.prototype` without the magic string.
- **External libraries**: jQuery `$.extend`, lodash `merge`, and similar recursive merges historically carry known PP gadgets — check bundled libs' versions.
- **Via browser APIs**: `fetch(url, options)` and `Object.defineProperty()` consume option objects; polluting a property they read (`headers`, `body`, `method`, `credentials`) turns a benign call into a gadget — no app code needed.
- **Goal on the client**: usually DOM XSS — pollute a property that lands in a script URL, `innerHTML`, or eval-like sink.

## Server-side prototype pollution (Node.js)

Harder to detect — you can't inspect `Object.prototype` remotely. PortSwigger's non-invasive detection primitives:

- **Polluted property reflection**: a property you polluted shows up in a response object or error.
- **Status code override**: `__proto__.status` polluted → Express apps return your status code.
- **JSON spaces override**: `__proto__.jsonSpaces` (used by Express `JSON.stringify` spacing) reflected in formatted output — a near-silent oracle.
- **Charset override**: polluted charset changes response encoding.
- **Scanning for sources**: try `__proto__` in JSON bodies, multipart fields, URL params — watch for any behavioral delta.
- **Filter bypass**: server-side key blocklists have the same gaps as client-side (constructor route, nesting, encoding).
- **RCE path**: Node's `child_process` reads options objects — `fork()`/`execSync()` honor inherited properties like `shell` (run command via a shell), `input` (stdin payload), `execArgv` (Node flags for the child), and `env` (e.g. injecting `NODE_OPTIONS=--require /tmp/x.js`). Polluting the inherited option a fork/exec call reads → command injection. Identify the request that reaches a `child_process` call, then pollute the option it consumes.

## Prevention knowledge (for remediation notes)

Sanitize keys on merge (blocklist `__proto__`, `prototype`, `constructor`); freeze prototypes (`Object.freeze(Object.prototype)`); create objects with `Object.create(null)` or `Map`; prefer library patches/config flags over hand-rolled filters.

## Lab reference

`https://portswigger.net/web-security/all-labs#prototype-pollution`
