# API Security

## Summary
- [Introduction](#introduction)
- [REST API](#rest-api)
- [RPC](#rpc)
- [SOAP](#soap)
- [GraphQL API](#graphql-api)

*Note: All image sources are from Bug Bounty Playbook v2*

## Introduction

En el pasado, las aplicaciones se creaban con un solo lenguaje, como PHP, pero la arquitectura de las aplicaciones actuales tiende a verse un poco diferente. La mayoría de las aplicaciones modernas se dividen en dos secciones:

- Front-end
- Back-end

<img src="https://user-images.githubusercontent.com/43796175/122650971-70a28580-d0fb-11eb-8f1d-e935bc0a879d.jpg">

La aplicación se divide en código de `Front-end` y `Back-end`. 

El `Front-end`, es la interfaz de usuario web que ve en su navegador, esto generalmente se escribe en un Framework de JavaScript moderno como `ReactJS` o `AngularJS`. 

El `Back-end` es la `API` y se puede escribir en varios idiomas:

<img src="https://user-images.githubusercontent.com/43796175/122651102-443b3900-d0fc-11eb-8ccb-56a5cd81ef83.jpg">

Cuando se trata de este tipo de aplicaciones, hay ciertas cosas que necesita saber y familiarizarse si quiere tener éxito. 

Hay varios tipos de `API` y cada una es ligeramente diferente, por lo que antes de comenzar a hackear la `API`, debe comprender algunas cosas.

## REST API

Si observa que una aplicación habla con una API de backend 9/10 veces, será una API REST. 

Un ejemplo de solicitud en Burp a una API REST podría parecerse a la siguiente imagen:

<img src="https://user-images.githubusercontent.com/43796175/122651291-78fbc000-d0fd-11eb-88ea-ed8485d3d149.jpg">

Hay varias maneras de reconocer una API REST:

- Mirando la request, el primer signo que indica que se trata de una solicitud de una API REST es el hecho de que los datos de la solicitud son una cadena JSON. Las cadenas de JSON son ampliamente utilizadas por las API REST. 
- El otro signo es que la aplicación está enviando una solicitud PUT. El método PUT es uno de varios métodos HTTP asociados con las API REST.
- Otra señal de que está tratando con una API REST es cuando la respuesta HTTP contiene un tipo MIME de JSON 

<img src="https://user-images.githubusercontent.com/43796175/122651935-86b34480-d101-11eb-8532-1ed3026c34b6.jpg">


## RPC

El archivo `xmlrpc.php` indica que existe una `API RPC`. 

`XMLRPC` utiliza `XML`, mientras que `JSONRPC` utiliza `JSON` para su tipo de codificación. 

Si el endpoint fuera una `API JSONRPC`, los datos estarían contenidos en una cadena `JSON` en lugar de un documento `XML`, esa es realmente la única diferencia entre los dos
`API RPC`.

Las `API REST` utilizan varios métodos HTTP, como `PUT`, `POST` y `DELETE`, pero las `API RPC` solo utilizan dos métodos:
- `GET`
- `POST`

Entonces, si ve una solicitud HTTP utilizando algo diferente a una solicitud `GET` o `POST`, sabrá que probablemente no sea una API RPC.


## SOAP

<img src="https://user-images.githubusercontent.com/43796175/122705461-6677a880-d21b-11eb-9874-fe6a75168754.jpg">

Como puede ver, el mensaje se envuelve primero en una etiqueta `<soapenv: Envelope>` que contiene las etiquetas de encabezado y cuerpo. 

Este valor se puede utilizar como indicador de que se trata de una `API SOAP`, así que esté atento a esta cadena. 

La parte del `header` es opcional y se usa para contener valores relacionados con la autenticación, tipos complejos y otra información sobre el mensaje en sí. 

El `body` es la parte del documento XML que realmente contiene nuestro mensaje como se muestra a continuación:
```
<soapenv:Body>
  <web:GetCitiesByCountry>
    <!--type: string-->
    <web:CountryName>argentina</web:CountryName>
  </web:GetCitiesByCountry>
<soapenv:Body>
```

Como puede ver en el `body` SOAP anterior, estamos llamando a un método llamado `GetCitiesByCountry` y pasando un argumento llamado `CountryName` con un valor de cadena de `argentina`.


## GraphQL API

`GraphQL` es un lenguaje de consulta de datos desarrollado por Facebook y lanzado en 2015. 

`GraphQL` actúa como una alternativa a la `API REST`. 

Las `API REST` requieren que el cliente envíe múltiples request a diferentes `endpoints` en la `API` para consultar datos de la base de datos de `backend`. Con `graphQL`, solo necesita enviar una request para consultar el `backend`. 

Esto es un poco más simple porque no tiene que enviar múltiples request a la `API`, se puede usar una sola solicitud para recopilar toda la información necesaria.

Por defecto, `GraphQL` no implementa autenticación, debe hacerlo el desarrollador. 

Esto significa que, por defecto, `GraphQL` permite que cualquiera realice consultas sobre información sensible, y estará disponible para los atacantes no autenticados.

Cuando realice fuzzing, busque por estos `endpoints` para encontrar instancias de `graphQL`:
```
/graphql
/graphiql
/graphql.php
/graphql/console
```

Una vez que encuentra una instancia, necesita averiguar qué consultas soporta. Esto lo puede averiguar usando el sistema de [Instrospection](https://graphql.org/learn/introspection/).

Si realiza la siguiente request, le mostrará todas las consultas que están disponibles en el punto final:
```
https://vulnerable.com/graphql?query={__schema{types{name,fields{name}}}}
```
<img src="https://user-images.githubusercontent.com/43796175/122706194-fcf89980-d21c-11eb-9f34-36cb696edcff.jpg">

Puede observar que hay un `type` llamado `User` y tiene dos campos denominados `username` y `password`.

Los `types` que comienzan con `__example__` deben ser ignorados, ya que son parte de un estándar.

Una vez que se encuentra un `type` interesante, puede consultar sus valores emitiendo la siguiente consulta:
```
http://example.com/graphql?query={TYPE_1{FIELD_1,FIELD_2} }
```

<img src="https://user-images.githubusercontent.com/43796175/122706465-9de75480-d21d-11eb-8fba-40f8a1a1fa3c.jpg">

Una vez enviada la consulta, obtendrá la información relevante y le devolverá los resultados. En este caso, obtenemos un conjunto de credenciales que se pueden usar para acceder a la aplicación.

`GraphQL` es una tecnología relativamente nueva que está empezando a ganar terreno entre las empresas emergentes y las grandes corporaciones. 

Aparte de la vulnerabilidad con la falta de autenticación por defecto, los `endpoints` de `graphQL` pueden ser vulnerables a otros errores como `IDOR`.








