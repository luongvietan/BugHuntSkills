# Ch16: Attacking Native Compiled Applications

Source: Chapter 16. When the app or its extensions are native code (C/C++ ISAPI filters, CGI, server modules, thick-client hybrids), the vuln classes are memory-safety bugs — same black-box reachability rules apply: you can only test what a request can reach.

## Bug classes — mechanism & remote detection

- **Stack buffer overflow**: unbounded copy into a fixed stack buffer (strcpy/sprintf/gets-style) → smash return address → RCE. Black-box probe: send inputs **progressively longer** than expected — crashes, 500s, or connection drops near a length boundary. String params, headers, filenames, cookies — anything parsed natively.
- **Heap overflow**: overwrites adjacent heap structures/control data → harder to exploit remotely but still corrupts state/RCE; same length-boundary fuzzing applies (target allocation-size boundaries — powers of 2, header sizes).
- **Off-by-one**: one byte past the buffer (loop bound error, null terminator miscount) → subtle crashes at exact boundary lengths — fuzz lengths *around* any apparent limit (N−1, N, N+1, N+2).
- **Integer overflows**: arithmetic wraps (`size+1` wraps to 0 → tiny alloc → huge copy). Probe: boundary integers — `0x7fffffff`, `0x80000000`, `0xffffffff`, near-limit negatives, `+1`/`-1` at limits — in any numeric param (length, count, id, quantity).
- **Signedness errors**: signed int compared/used as unsigned (or vice versa) → negative value becomes huge positive. Probe: `-1`, `-0x8000…` in size/count fields.
- **Format strings**: input reaches `printf(user_input)` → `%x %s %n` read/write memory. Probe: `%x%x%x%x`, `%s%s%s`, `%n` — watch for hex dumps of stack in response or crashes (`%n` writes → crash = strong signal).

## Black-box hunting

1. Identify native surface: server modules (ISAPI filters/extensions, Apache modules), CGI binaries, custom services behind web front-ends, upload processors, image/PDF parsers — the components that *parse* attacker data are the targets.
2. Length fuzzing at boundaries: powers of two, allocation units, observed limits — binary-search the crash point.
3. Integer boundary values in every numeric field; negative/hex/decimal forms.
4. Format specifiers in every string field; watch for `%x`-style leaks (stack data in response = format-string read primitive confirmed).
5. Crash analysis: repeatable crash at a controlled length/value is the finding — document input, boundary, and observed behavior. Further exploitation (control of EIP/heap) is typically out of black-box scope but the crash alone demonstrates the defect.

## Where they hide

- Legacy components that predated managed frameworks and never got rewritten; URL-rewriting and ISAPI filters (they parse *every* request incl. headers/methods/URIs); file-upload handlers and document parsers (complex binary formats = parser bugs); custom authentication modules.

## Checklist

- [ ] Native parsing surface mapped (filters, modules, CGIs, upload processors).
- [ ] Length boundary fuzzed on reachable string fields; crash boundary identified.
- [ ] Integer boundary values tried (signed/unsigned, ±1 around limits).
- [ ] Format specifiers submitted; stack-leak or crash observed.
- [ ] Repeatable crash documented as the PoC — no weaponization needed for authorized test.
