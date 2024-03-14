# Transformaciones

Siempre será el caso que hay que hacer ajustes en los queries de algún sistema
para que se acoplen a los requerimientos de nimbox o a las necesidades del
negocio. Hay varios mecanismos de transformación que se pueden utilizar. Estos
mecanismos se listan a continuación en el orden en que son más deseables.

Trabajar con estos mecanismos requiere ver el efecto de las configuraciones en
el query final. El comando de línea `conx` permite ver el query que será
utilizado para hacer la lectura de la base de datos. Solo hay que ejecutar el
comando `conx sql query Product`. Esto mostrará el query final que se utilizará
para hacer la lectura de la tabla `Product` en la base de datos.

## Substitución de valores

Es posible incluir mecanismos de substitución de valores en los queries. La
substitución se hace a través de un motor de plantillas. Por ahora estamos
utilizando [FreeMarker](https://freemarker.apache.org/), pero es posible añadir
otros.

Para hacer una substitución hay que hacer dos cosas: incorporar la información
de substitución en la configuración del conector y ajustar el query para
utilizar los valores de substitución. 

La información de substitución se incluye en la configuración del conector. Hay
que identificar el motor de plantillas (e.g., `"renderer": "FREEMARKER"`) y
añadir los valores (en este caso `defaultPriceListId`), para que estén
disponible en el proceso de substitución. La configuración se vería así:

```json
{
    "renderer": "FREEMARKER",
    "database": {
        "substitutions": {
            "defaultPriceListId": "2"
        }
    }
}
```

Este cambio indica que queremos utilizar el motor de plantillas FreeMarker y que
queremos que esté disponible el valor `defaultPriceListId` con el valor `2`. En
la parte de substitución se pueden incluir tantos valores como se necesiten y
hasta estructuras completas de `json`. Lo importante es que los valores sean
procesados correctamente por FreeMarker.

En el query se incluye el valor de substitución de la siguiente forma:

```sql
SELECT
    P.ItemCode AS "originalId",
    COALESCE(PL.Price, 0.0) AS "price",
FROM
    OITM P LEFT JOIN ITM1 PL 
        ON P.ItemCode = PL.ItemCode AND 
            PL.PriceList = ${substitutions.defaultPriceListId!'1'}
```

Los cambios el query tienen que estar hechos con la sintaxis de FreeMarker. Hay
cientos de comandos y formas de hacer los cambios lo que permite tener mucha
flexibilidad en el momento de hacer los cambios.

El SQL final sería:

```sql
SELECT
    P.ItemCode AS "originalId",
    COALESCE(PL.Price, 0.0) AS "price"
FROM
    OITM P LEFT JOIN ITM1 PL 
        ON P.ItemCode = PL.ItemCode AND 
            PL.PriceList = 2
```

Free marker se encarga de hacer la substitución de
`${substitutions.defaultPriceListId!'1'}` por el valor de `defaultPriceListId`
si este existe, o por `1` si no existe (esto es lo que hace el `!` en el query).
Esto permite que no sea necesario incluir el valor en la configuración si no se
quiere cambiar el valor por defecto.

## Substitución de columnas

Los cambios de columnas son más complicados que los cambios de valores porque
requieren analizar el SQL. Este proceso de análisis se hace con un motor de
análisis de SQL. Por ahora estamos utilizando
[jSQL](https://jsqlparser.github.io/JSqlParser) para hacer el análisis y la
substitución, pero es posible añadir otros.

En esta caso hay que identificar el motor de substitución de SQL (e.g.,
`"replacer": "JSQLPARSER"`) e incluir la substitución de la columna en el
archivo de configuración.

```json
{
    "replacer": "JSQLPARSER",
    "database": {
        "columns": {
            "Product": {
                "comment": "P.ItemCode + ':' + P.ItemName",
            }
        }
    }
}
```

Este cambio indica que queremos que la columna `comment` del query `Product` sea
sustituida por `P.ItemCode + ':' + P.ItemName`. El analizador de SQL se encarga
de hacer la substitución de la columna `comment` por `P.ItemCode + ':' +
P.ItemName` en el query sin tener que realizar una substitución de valores.

Asumiendo que el SQL original es:

```sql
SELECT
    P.ItemCode AS "originalId",
    NULL AS "comment"
FROM
    OITM P
```

El sql final sería:

```sql
SELECT
    P.ItemCode AS "originalId",
    P.ItemCode + ':' + P.ItemName AS "comment"
FROM
    OITM P
```

Para hacer estos cambios, el analizador de SQL tiene que ser "parsear" el sql,
cambiar la columna y volver a "renderizar" el sql. Este proceso hace que el
formato original del SQL se pierda, pero se garantiza que el SQL final sea
válido.

Es posible mezclar la substitución de valores con la substitución de columnas.
En este caso, el motor de plantillas se encarga de hacer primero la substitución
de valores y el analizador de SQL se encarga de hacer la substitución de
columnas.

Con una configuración así:

```json
{
    "renderer": "FREEMARKER",
    "replacer": "JSQLPARSER",
    "database": {
        "columns": {
            "Product": {
                "comment": "P.ItemCode + '${substitutions.separator!':'}' + P.ItemName"
            }
        },
        "substitutions": {
            "separator": "-"
        }
    }
}
```

El sql final sería:

```sql
SELECT
    P.ItemCode AS "originalId",
    P.ItemCode + '-' + P.ItemName AS "comment"
FROM
    OITM P
```

Free marker se encarga de hacer la substitución de
`${substitutions.separator!':'}` por el valor de `separator` si este existe, o
por `:` si no existe. Luego, el analizador de SQL se encarga de hacer la
substitución de la columna `comment` por `P.ItemCode + '-' + P.ItemName` en el
query.

## Substitución de tablas

Los cambios de tablas son la peor opción. Esto se debe a que los cambios de
tablas requieren un análisis más profundo del SQL original para estar seguros
que el contrato original se mantiene. 

En este caso se requiere incluir el query completo en la configuración.

```json
{
    "database": {
        "tables": {
            "Product": "SELECT P.ItemCode AS \"originalId\", P.ItemName AS \"name\" FROM OITM P"
        }
    }
}
```

Este cambio indica que queremos que el query `Product` sea sustituido por el
provisto en la configuración. De esta forma, independientemente de cuál es el
query por defecto, el query final será el provisto en la configuración.

El sql final sería:

```sql
SELECT
    P.ItemCode AS "originalId",
    P.ItemName AS "name"
FROM
    OITM P
```

Es posible mezclar la substitución de valores con la substitución de tablas. En
este caso, el motor de plantillas se encarga de hacer primero la substitución de
valores y el resultado es el que se utiliza para realizar el query. En este caso
no se utiliza el analizador de SQL.
