# IDOR

Las referencias directas a objetos inseguras se producen cuando una aplicación proporciona acceso directo a objetos en función de la entrada proporcionada por el usuario. Como resultado de esta vulnerabilidad, los atacantes pueden eludir la autorización y acceder a los recursos del sistema directamente, por ejemplo, registros o archivos de bases de datos.

## Summary 

* [Tools](#tools)
* [Exploit](#exploit)
* [Lugares Poco Comunes](#lugares-poco-comunes)
* [IDs Hasheados](#ids-hasheados)
* [Adivinar IDs](#adivinar-ids)
* [HPP con IDOR](#hpp-con-idor)
* [Blind IDORs](#blind-idors)
* [Cambiar el Método de la Request](#cambiar-el-metodo-de-la-request)
* [Cambiar el Tipo de Archivo Solicitado](#cambiar-el-tipo-de-archivo-solicitado)
* [Envolver ID en un Array](#envolver-id-en-un-array)
* [Cómo Aumentar el Impacto de las IDOR](#como-aumentar-el-impacto-de-las-idors)
* [Examples](#examples)



## Tools

- Burp Suite plugin Authz
- Burp Suite plugin AuthMatrix
- Burp Suite plugin Authorize


## Exploit

<img src="https://user-images.githubusercontent.com/43796175/115991665-326f6680-a58f-11eb-8b3a-117cf32d8710.png"></img>

The value of a parameter is used directly to retrieve a database record.

```powershell
http://foo.bar/somepage?invoice=12345
```

The value of a parameter is used directly to perform an operation in the system

```powershell
http://foo.bar/changepassword?user=someuser
```

The value of a parameter is used directly to retrieve a file system resource

```powershell
http://foo.bar/showImage?img=img00011
```

The value of a parameter is used directly to access application functionality

```powershell
http://foo.bar/accessPage?menuitem=12
```

## Lugares Poco Comunes

No ignore los ID codificados y con hash.

Cuando se enfrenta a un ID codificado, es posible decodificar el ID utilizando esquemas de codificación comunes.

Y si la aplicación utiliza una identificación hash aleatoria, vea si la identificación es predecible. 

A veces, las aplicaciones utilizan algoritmos que producen una entropía insuficiente y, como tal, los ID se pueden predecir después de un análisis cuidadoso. 

En este caso, intente crear algunas cuentas para analizar cómo se crean estos ID. Es posible que pueda encontrar un patrón que le permita predecir las ID que pertenecen a otros usuarios.

Si los ID de referencia de objeto parecen impredecibles, vea si hay algo que pueda hacer para manipular el proceso de creación o vinculación de estos ID de objeto.

## IDs Hasheados

No siempre es tan fácil como cambiar su `ID` de usuario por otro `ID`, a veces es no puede adivinar el `ID` asociado, como se muestra a continuación:
```
POST /challenge/users/vfd3f0jkui4555kJNJHahh023
Host: vulnerable.com

userID=8f14e45fceea167a5a36dedd4bea2543
```

Observando el ID del user `8f14e45fceea167a5a36dedd4bea2543`, podría pensar que es una identificación aleatoria que es imposible adivinar, pero puede que no sea el caso. 

Es una práctica común [HASHEAR](https://www.md5online.org/md5-decrypt.html) el `ID` de los usarios antes de almacenarlos en una base de datos, así que tal vez eso sea lo que está sucediendo aquí:

<img src="https://user-images.githubusercontent.com/43796175/122650815-85324e00-d0fa-11eb-89d8-b7d63db9120a.jpg">

Luego, hashee los números correspondientes para obtener su IDOR.

## Adivinar IDs

Si no se utilizan ID en la solicitud generada por la aplicación, intente agregarlo a la solicitud. Intente agregar `id`, `user_id`, `message_id` u otros parámetros de referencia de objeto y vea si hace una diferencia en el comportamiento de la aplicación.

Por ejemplo, si esta solicitud muestra todos sus mensajes directos:

```
GET /api_v1/messages
```

Entonces, ¿esta request mostraría los mensajes de otro usuario?

```
GET /api_v1/messages?user_id=ANOTHER_USERS_ID
```

## HPP con IDOR

Las vulnerabilidades de HPP (que proporcionan múltiples valores para el mismo parámetro) también pueden conducir a IDOR. 

Es posible que las aplicaciones no anticipen que el usuario envíe varios valores para el mismo parámetro y, al hacerlo, puede bypassear el control de acceso establecido en el endpoint.

Se vería así. Si esta solicitud falla:

```
GET /api_v1/messages?user_id=[ANOTHER_USERS_ID]
```

Prueba esto:

```
GET /api_v1/messages?user_id=[YOUR_USER_ID]&user_id=[ANOTHER_USERS_ID]
```

O esto:

```
GET /api_v1/messages?user_id=[ANOTHER_USERS_ID]&user_id=[YOUR_USER_ID]
```

O proporcione los parámetros como una lista:

```
GET /api_v1/messages?user_ids[]=[YOUR_USER_ID]&user_ids[]=[ANOTHER_USERS_ID]
```

## Blind IDOR

A veces, los endpoints susceptibles a IDOR no responden directamente con la información filtrada. En su lugar, podrían llevar a la aplicación a filtrar información en otro lugar: en archivos de exportación, correos electrónicos y tal vez incluso alertas de texto.

## Cambiar el Método de la Request

Si un método de solicitud no funciona, hay muchos otros que puede probar en su lugar: GET, POST, PUT, DELETE, PATCH ...

Un truco común que funciona es sustituir POST por PUT o viceversa: 
¡es posible que no se hayan implementado los mismos controles de acceso!

## Cambiar el Tipo de Archivo Solicitado

A veces, cambiar el tipo de archivo del fichero solicitado puede llevar a que el servidor procese la autorización de manera diferente. Por ejemplo, intente agregar `.json` al final de la URL de solicitud y vea qué sucede.

## Envolver ID en un Array

```
{“id”:111} --> 401 Unauthriozied
{“id”:[111]} --> 200 OK
```

```
{“id”:111} --> 401 Unauthriozied
{“id”:{“id”:111}} --> 200 OK
```

## Cómo Aumentar el Impacto de las IDOR

### IDOR Críticas Primero

Siempre busque IDOR en funcionalidades críticas primero. 

Los IDOR basados en lectura y escritura pueden tener un gran impacto.

En términos de IDOR de cambio de estado (escritura), restablecimiento de contraseña, cambio de contraseña, IDOR de recuperación de cuenta a menudo tienen el mayor impacto comercial. (Digamos, en comparación con un IDOR de "cambiar la configuración de suscripción de correo electrónico").

En cuanto a los IDOR que no cambian de estado (lectura), busque funcionalidades que manejen la información confidencial en la aplicación. 

Por ejemplo, busque funcionalidades que manejen mensajes directos, información confidencial del usuario y contenido privado. Considere qué funcionalidades de la aplicación utilizan esta información y busque los IDOR en consecuencia.


## Examples

* [HackerOne - IDOR to view User Order Information - meals](https://hackerone.com/reports/287789)
* [HackerOne - IDOR on HackerOne Feedback Review - japz](https://hackerone.com/reports/262661)
