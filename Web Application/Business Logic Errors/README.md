# Business Logic Errors

Las vulnerabilidades de lógica empresarial son fallas en el diseño y la implementación de una aplicación que permiten a un atacante provocar un comportamiento no deseado. 

Esto potencialmente permite a los atacantes manipular la funcionalidad legítima para lograr un objetivo malicioso. 

Estas fallas son generalmente el resultado de no anticipar estados de aplicación inusuales que pueden ocurrir y, en consecuencia, no manejarlos de manera segura.

## Summary

* [Manipulación de Fechas](#manipulacion-de-fechas)
* [Manipulación de Pagos](#manipulacion-de-pagos)
* [Manipulación de Cantidades](#manipulacion-de-cantidades)
* [Manejo Inconsistente de Entrada de Usuario](#manejo-inconsistente-de-entrada-de-usuario)
	* [Reasignación de Email](#reasigacion-de-email)
	* [Sobreescritura de Email](#sobreescritura-de-email)
	* [Eliminar Parámetros](#eliminar-parametros)
* [Bypass Workflow de Compra](#bypass-workflow-de-compra)
* [Defectos Específicos de la Aplicación](#defectos-especificos-de-la-aplicacion)
	* [Dos o más cupones de descuento](#dos-o-mas-cupones-de-descuento)


## Manipulacion de Fechas

Cuando se modifica el rango de "búsqueda" en algún campo interesante de la aplicación, esto puede conducir a un DoS, divulgación de información sensible, etc.

## Manipulacion de Pagos

Por ejemplo, un tipo de datos numérico puede aceptar valores negativos. Dependiendo de la funcionalidad relacionada, puede que no tenga sentido que la lógica empresarial lo permita. Sin embargo, si la aplicación no realiza una validación adecuada del lado del servidor y rechaza esta entrada, un atacante puede pasar un valor negativo e inducir un comportamiento no deseado.

## Manipulación de Cantidades

Tomemos el ejemplo sencillo de una tienda online. Al pedir productos, los usuarios suelen especificar la cantidad que quieren pedir. Aunque cualquier número entero es teóricamente una entrada válida, la lógica empresarial podría evitar que los usuarios pidan más unidades de las que hay actualmente en stock, por ejemplo.

Agregar una cantidad excesiva de productos, para superar el valor límite de 2,147,483,647 soportado por el backend. Verá un valor negativo devuelto por la aplicación. Debe ajustar los valores finales para que se pueda comprar un producto que, a priori, no tiene saldos suficientes. Utilizando "Intruder" con "null payloads" puede ocasionar la "agregación" indefinida de cantidades para desencadenar este comportamiento.

## Manejo Inconsistente de Entrada de Usuario

### Reasignación de Email

1. Se debe enviar un "Registo" de usuario con el mail extenso, que supere los 255 carácteres.

```
AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA[youremail]@[legitimo_domain.com]
```

2. Revisar el inbox para confirmar que haya llegado con exito la solicitud de registro.

3. Verificar que efectivamente se "Trunca" el email asociado a la cuenta registrada.

4. Registrar un nuevo usuario, pero esta vez los primeros 255 carácteres deben concluir con la letra "m" de ".com", con el objetivo de suplantar la dirección legítima de la empresa víctima y registrar un usuario malicioso que sea parte de "dominio_victima":

```
AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAyouremail@[dominio_victima].[legitimo_domain.com]
```

### Sobreescritura de Email

User 01: Admin
```
admin@[domain_company].com
```

User 02: Atacante
```
atacante@email.com
```

Cambie el email del atacante por el del usuario administrador.


### Eliminar Parámetros

Debe intentar eliminar cada parámetro por turno y observar qué efecto tiene esto en la respuesta. Debes asegurarte de:

- Solo elimine un parámetro a la vez para asegurarse de que se alcancen todas las rutas de código relevantes.

- Intente eliminar el nombre del parámetro y el valor. El servidor normalmente manejará ambos casos de manera diferente.

- Siga los procesos de varias etapas hasta su finalización. A veces, alterar un parámetro en un paso tendrá un efecto en otro paso más adelante en el flujo de trabajo.

Esto se aplica tanto a la URL como a los POSTparámetros, pero no olvide comprobar también las cookies. Este simple proceso puede revelar algún comportamiento extraño de la aplicación que puede ser explotable.

La siguiente petición dará un error en "current-password":

```html
username=cgonzalo&current-password=test&new-password-1=asd&new-password-2=asd
```

Se borra el parámetro "current-password" y la petición se acepta igualmente:

```html
username=cgonzalo&new-password-1=asd&new-password-2=asd
```

Ahora cambie el nombre de usuario:

```html
username=administrator&new-password-1=asd&new-password-2=asd
```

## Bypass Workflow de Compra

La idea es pasar por alto las request donde se "comprueba" el saldo para comprar determinado artículo.

Agregar al carrito lo que se desea comprar.

La siguiente request verifica el saldo:

```html
POST /cart/checkout
```

La siguiente request confirma y realiza el pedido:

```html
GET /cart/order-confirmation?order-confirmation=true
```

Si se agrega un artículo y luego se envía la última request, no se verifica el precio y el artículo se compra sin saldo.


## Defectos Específicos de la Aplicación

En muchos casos, encontrará fallas lógicas que son específicas del dominio comercial o del propósito del sitio.

La funcionalidad de descuento de las tiendas en línea es una superficie de ataque clásica cuando se busca fallas lógicas. Esto puede ser una mina de oro potencial para un atacante, con todo tipo de fallas lógicas básicas que ocurren en la forma en que se aplican los descuentos.

Por ejemplo, considere una tienda en línea que ofrece un 10% de descuento en pedidos superiores a $ 1000. Esto podría ser vulnerable a abusos si la lógica empresarial no verifica si el pedido se cambió después de aplicar el descuento. En este caso, un atacante podría simplemente agregar artículos a su carrito hasta que alcancen el umbral de $ 1000, luego eliminar los artículos que no desea antes de realizar el pedido. Luego recibirían el descuento en su pedido aunque ya no satisfaga los criterios previstos.

### Dos o más cupones de descuento

Utilizar dos o más cupones de descuento de manera alterada, puede permitirle agregar infinitas instancias de cada uno de ellos, siempre y cuando se alternen entre ellos.

```txt
Ingrese descuento: CUPON01
Ingrese descuento: CUPON02
Ingrese descuento: CUPON01
Ingrese descuento: CUPON02
Ingrese descuento: CUPON01
Ingrese descuento: CUPON02
Ingrese descuento: CUPON01
Ingrese descuento: CUPON02
Ingrese descuento: CUPON01
Ingrese descuento: CUPON02
```

