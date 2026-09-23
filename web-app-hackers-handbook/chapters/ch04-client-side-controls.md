# Ch5: Bypassing Client-Side Controls

Source: Chapter 5. Applications trust the client in two ways: they **transmit data via the client** (assuming it returns unmodified) and they **validate on the client** (assuming enforcement happened). Both assumptions are false — the user controls the whole client, and every check can be observed, replicated, or skipped entirely.

## Mechanism → where data rides the client

- **HTML form fields**: hidden inputs (`price=`, `productid=`, `discount=`), disabled fields, `maxlength` — all trivially modifiable at the proxy. Disabled/hidden only means "not rendered editable," not "not submitted."
- **Cookies**: apps ship logic-bearing values in cookies — `role`, `uid`, `agreed`, prices, serialized state. Tamper at the browser/proxy.
- **URL parameters**: preset values, IDs, return URLs.
- **Referer header**: used to mark "came from step N" or "came from admin area" — forgeable; also used as an access-control or CSRF check (weak by design).
- **Opaque data**: encrypted/signed blobs, ViewState-style serialization. Base64-decode everything opaque; if it's encrypted, look for an *encryption oracle* (ch10 logic ex.1) or try replay/substitution from elsewhere in the app; if signed, test whether the signature is actually verified (remove/reorder fields).
- **Remoting & serialized state**: AMF, custom blobs — decode, tamper, re-encode.

## Client-side validation → always bypassable

- **Script validation**: `onsubmit`/`onblur` JS checks, per-input rules. Read the JS to learn the rules (it's free documentation of server expectations), then submit via proxy, bypassing the browser entirely. "Validation only happens in the browser" is the bug.
- **Client extensions (Java/Flash/Silverlight)**: bytecode decompiles (JD-GUI, JAD, flasm, SWFScan). Extension performs crypto/signing → extract keys/logic, or intercept *before* the extension (upstream proxy or browser hooks) and submit raw.
- **Obfuscated/encrypted client code**: obfuscation ≠ security; deobfuscate or instrument — the check still runs on the attacker's machine.

## Attack method

1. Capture every request; flag every client-supplied value the server *uses* (not just displays).
2. For each: modify value, omit the parameter entirely (name *and* value — servers treat absent ≠ empty differently), reorder duplicates.
3. Disable/bypass each client control: proxy for forms, decompile for extensions, JS console for script checks.
4. Where the server *did* validate: find the check's representation gap (encoding, type confusion, negative/overflow for numeric).

## Checklist

- [ ] Every hidden/disabled form field tested for tampering and for server-side revalidation.
- [ ] Every cookie and URL param modified, deleted, duplicated.
- [ ] Opaque blobs decoded; signature/encryption verification probed (replay, cross-feature substitution).
- [ ] Each JS validation rule catalogued → same input submitted through proxy without the rule.
- [ ] Client extensions decompiled or bypassed at the network layer.
