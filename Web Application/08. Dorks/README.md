## Summary

- [Google Dorks](#google-dorks)
- [Github Dorks](#github-dorks)
- [Shodan](#shodan)
  - [ESI Detection](#esi-detection)

## Google Dorks:

### Sensitive Information

Here are Top 5 Google dorks for identifying interesting and potentially sensitive information about your target (example.com)
```
inurl:example.com intitle:"index of"
```
```
inurl:example.com intitle:"index of /" "*key.pem"
```
```
inurl:example.com ext:log
```
```
inurl:example.com intitle:"index of" ext:sql|xls|xml|json|csv
```
```
inurl:example.com "MYSQL_ROOT_PASSWORD:" ext:env OR ext:yml -git
```
```
example.com "password"
```

With these dorks we are looking for open directory listing, log files, private keys, spreadsheets, database files and other interesting data.

### Kibana

Kibana will return a content length of 217 if it is publicly open and one can access the dashboard without authentication:

```
inurl:app/kibana
inurl:app/kibana intext:Loading Kibana
inurl::5601/app/kibana
kibana content-lenght: 217 (in Shodan search)
```

## Github Dorks:

```
domain.com "password"
```

## Shodan

### ESI Detection

Detectar ESI (edge-side-include-injection), en el  encabezado HTTP `Surrogate-Control`. Este encabezado se utiliza para indicar a los servidores Proxies que las  etiquetas ESI podrían estar presentes en la respuesta y que deben analizarse como tales:
```
Surrogate-Control: content="ESI/1.0"
```
