# Fuzzing

## Summary
- [Lists](#lists)
  - [users and passwords](#users-and-passwords)
  - [seclist](#seclist)
  - [Intruder Attacks](#intruder-attacks)
-   [Dirb](#dirb)
-   [ffuf](#ffuz)



## Lists

### Users and Passwords

https://github.com/danielmiessler/SecLists/tree/master/Passwords/Default-Credentials

### SecList:

https://github.com/danielmiessler/SecLists

### Intruder Attacks

- Sniper: Utiliza una única lista de carga útil; Reemplaza una posición a la vez;
- Battering RAM: Utiliza una única lista de carga útil; Reemplaza todas las posiciones al mismo tiempo
- Pitchfork: Cada posición tiene una lista de carga útil correspondiente; Entonces si hay dos posiciones a modificar, cada una obtiene su propia lista de carga útil.
- Cluster Bomb: Utiliza cada lista de carga útil y neumáticos diferentes combinaciones para cada posición.

## Dirb
```
dirb https://vulnerable.com/ /your/wordlist/path/  
```

## ffuz

### Con extensiones
```sh
ffuf -w g0ld3n.txt -u https://vulnerable.com/FUZZ -e .txt,.php,.sh,.py,.aspx,.asp,.php
```

### Con subdirectorios y una profundidad de 2 subdirectorios para buscar
```sh
ffuf -w g0ld3n.txt -u https://vulnerable.com/FUZZ -e .txt,.php,.sh,.py,.aspx,.asp,.php -recursion -recursion-depth 2
```

### Descubrir subdominios

Esta herramienta es capaz de encontrar subdominios sin registros DNS a velocidades ultrarrápidas.

La herramienta utiliza el encabezado de `Host` en una solicitud HTTP para buscar subdominios. 

La bandera `-H` se usa para especificar encabezados de solicitud HTTP. Tenga en cuenta que se permiten varias banderas -H.
```
ffuf -w subdomains.txt -u https://vulnerable.com/ -H "Host: FUZZ.vulnerable.com"
```

Si la herramienta proporciona muchos subdominios como salida y la mayoría de ellos no están presentes en la realidad, se pueden utilizar las opciones de filtro que ofrece la herramienta.

### Cambio de Método HTTP
```
ffuf -w wordlist_api/g0ld3n-api.txt -u https://vulnerable.com/FUZZ -e .txt,.php,.sh,.py,.aspx,.asp -recursion --fw=51 -recursion-depth 2 -H 'Authorization: Bearer JWT' -X POST
```

### Discover the Parameter of a Shell in WebServer

```
ffuf -c -w /usr/share/seclists/Discovery/Web-Content/burp-parameter-names.txt -u http://sec03.rentahacker.htb/shell.php?FUZZ=id
```

*Reference by [Scavenger from Hack the Box](#https://0xdf.gitlab.io/2020/02/29/htb-scavenger.html)*



