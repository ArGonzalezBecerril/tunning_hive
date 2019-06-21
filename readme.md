# Tunning hive
[![N|Solid](https://i.ibb.co/n3DTftq/hive-Tunning.png)](https://nodesource.com/products/nsolid)

El proceso para llevar a cabo el tunning en hive suele ser complejo y en muchas ocasiones es necesario improvisar configuraciones para establecer las bases de una configuración idónea.
A continuación se enumeran los siguientes tips para llegar a obtener: coherencia, consistencia, flexibilidad y sobre todo rapidez en las consultas hechas a hive.

- **Uso de particionado de datos**: Ya sea por mes, año o incluso por algún campo que represente un ciclo esta perfecto.
- **Buketing:** Básicamente representa los fragmentos o pedazos dispersados alrededor del cluster el algo similar al **repartition** que usamos cuando almacenamos en spark.
- **Uso de hints:** Si estamos realizando un famoso join pesado lo mejor será usar **Mapjoin** para cargar la tabla en memoria y realizar el join(Siempre y cuando el cluster no esté en un entorno compartido y sobre todo no existan jobs ejecutándose).
-  **Compresión:** La compresión es un tema importante recordemos que hive soporta avro, parquet y orc los cuales son los más conocidos cabe destacar que a parquet le podemos añadir un algoritmo de compression como **snappy o gzip**. ORC solo es recomendable en caso de que la tabla se usa mas como repositorio que para análisis.

>>Hints: Es una característica poco conocida de hive,  normalmente este concepto los hemos escuchado en motores relacionales como Oracle. Hive implementa este concepto y mediante palabras clave podemos hacer uso de ello. Por tal motivo un ejemplo es el uso de **Mapjoin**

**Mapjoin:** Permite cargar una tabla en memoria de manera muy rápida de esta forma se realiza el procesamiento en un solo paso sin necesidad de usar el proceso típico  Map-Reduce en realidad el numero de procesamiento se reduce de 2 a 1.

##### Ejemplo:

```sql
--Join de tabla MOVIES & MOVIES_RATING
select /*+ MAPJOIN(user) */ m.movie_name, mr.rating, u.name 
  FROM movie m 
  INNER JOIN movie_rating mr on m.movieid = mr.movieid
  INNER JOIN user u  on mr.userId = u.userId
  ORDER BY  mr.rating;
--Al ser una tabla de TB de datos es posible que notemos un ligero aumento de velocidad
```
#### Compresión
La compresión es otro concepto fundamental acerca de cómo la información puede ir distribuida de forma concisa y coherente.
De la siguiente forma podríamos establecer una compresión especifica:

```sql
--Crear la tabla movie_rating
create external table movie_rating(
    userId int,
    movieId int,
    rating double
)row format delimited
fields terminated by ",";
--De las siguientes líneas solo se debe elegir una:
stored as parquet  --En caso de usar parquet
stored as avro     --En caso de usar avro
stored as orc      --En caso de usar orc, tambien podriamos optar por usar sequencefile
```

#### Testing Mapjoin
Una vez activado el **mapjoin** podríamos obtener un top-ranking de películas con mejor calificación o score.
```sql
select movie_name, rating, name, rank() over (order by rating desc) as rank
     from (select /*+ MAPJOIN(user) */ m.movie_name, mr.rating, u.name 
           from movie m 
           inner join movie_rating mr on m.movieid = mr.movieid
           inner join user u  on mr.userId = u.userId
           order by mr.rating)s;
--Obteniendo como resultado lo siguiente:
--Podemos optar por un denserank para poder observar en secuencia el top-ten
```
>>Una limitante de esta operación es que no podremos convertir un full-outer-join a un join de mapa, sin embargo podemos convertir un **left-outer-join** a un join de mapa en hive **Siempre y cuando la tabla de la izquierda no supere los 25mb** o incluso pasa lo mismo con un **right-outer-join**. Es importante entender que los 25 mb es solo un valor default si deseamos modificar esta configuración es de la siguiente manera.

```sh
hive> set hive.mapjoin.smalltable.filesize = 3000000 
```
[![N|Solid](https://i.ibb.co/Thq9DMX/ranking.png)](https://nodesource.com/products/nsolid)

#### Particionado
Este concepto principalmente es aplicable a campos de tipo fecha para lograr establecer una periodicidad continua ya sea por mes o año incluso puede darse el caso que sea necesario particionar por día.

No es obligatorio elegir una columna de tipo fecha pero es la más concisa a la hora de establecer particionados periódicamente.

**Licence GPL**
Continua en una segunda parte...