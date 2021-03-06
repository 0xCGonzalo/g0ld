# Information Disclosure

La divulgación de información, también conocida como fuga de información, es cuando un sitio web revela involuntariamente información confidencial a sus usuarios. 

***

## Summary 

* [Archivos para Rastreadores Web](#archivos-para-rastreadores-web)
* [Herramientas de Burpsuite](#herramientas-de-burpsuite)
* [Source Code Disclosure via Backup Files](#source-code-disclosure-via-backup-files)
* [Métodos HTTP](#trace-http)
* [Change "Accept" Header](#change-accept-header)

***

## Archivos para Rastreadores Web

```
/robots.txt
/sitemap.xml
```

***

## Herramientas de Burpsuite

"Target --> SiteMap --> Engagement Tools"

***

## Source Code Disclosure via Backup Files

Cuando un servidor maneja archivos con una extensión en particular ".php", normalmente ejecutará el código, en lugar de simplemente enviarlo al cliente como texto. Sin embargo, en algunas situaciones, puede engañar a un sitio web para que devuelva el contenido del archivo.

```
/private.php --> 200 OK (no devuelve nada, solo ejecuta)
/private.php~ --> 200 OK (devuelve el fichero, ya que se está llamando al archivo temporal)
```

***

## Métodos HTTP

### TRACE

TRACE: Repite la solicitud entrante.

Es posible obtener información de encabezados sensibles con el método TRACE.

```
POST /admin --> 401 Unauthorized
TRACE /admin --> 200 OK
```

Esta última request devuelve un header oculto:

```
X-Custom-IP-Authorization: 181.61.60.225
```

Luego, agregar este header a cada solicitud que se envíe como:

```
X-Custom-IP-Authorization: 127.0.0.1
```

Y se accede como usuario Administrator.

### Otros Métodos para Generar Errores

```
HEAD (Solicita la lectura del encabezado de una página Web)
OPTION (Consulta ciertas opciones)
POST
GET
PUT (Solicita el almacenamiento de una página Web)
HELP 
DELETE (Elimina la página Web)
CONNECT (Reservado para uso futuro)
```

***

## Change "Accept" Header

Find information disclosure vulnerabilities in some web servers by changing the "Accept" header.

`Accept: application/json, text/javascript, */*; q=0.01`

***
