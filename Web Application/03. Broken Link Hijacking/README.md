# Broken Link Hijacking

## Summary

* [Exploitation](#exploitation)
* [Check Online](#check-online)
* [References](#references)

## Exploitation

Encuentre y clickee sobre links que aparezcan en la aplicación objetivo (Por ejemplo, links hacia redes sociales u otros)

Mientras hace esto, corra esta [herramienta](https://github.com/stevenvachon/broken-link-checker) para ayudarlo a encontrar enlaces externos rotos.

```
blc -rof --filter-level 3 https://example.com/
```

La salida en un caso de exito se verá así:

```
─BROKEN─ https://www.linkedin.com/company/ACME-inc-/ (HTTP_999)
```

Si, por ejemplo, algún enlace lo redirige a un link de LinkdIn que no exista (404 response), intente crear una cuenta en LinkedIn con el nombre del recurso externo llamado desde la aplicación víctima.

## Check Online

https://brokenlinkcheck.com/

https://ahrefs.com/broken-link-checker

## References

https://medium.com/@bathinivijaysimhareddy/how-i-takeover-the-companys-linkedin-page-790c9ed2b04d
