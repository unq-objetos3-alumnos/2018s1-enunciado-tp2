# Functional-relational mapper

Queremos desarrollar una librería en Scala que nos permita tratar con bases de 
datos relacionales.
Para abstraer la capa de persistencia, nos gustaria tratar las tablas como si 
fueran colecciones, cumpliendo:


 - Nuestras queries deben ser type-safe.
 - No agregar información de la capa de persistencia en el modelo del dominio.
 - Crear todas las abstracciones que sean necesarias.
 - Solo nos interesa persistir clases inmutables (case class).
 - Debo poder reutilizar definiciones de queries sin causar efectos de lado.
 - Se debe respetar la idea de transacción y permitir al usuario realizar varias operaciones dentro de una transacción.


## Backends

Pueden usar el motor de bases de datos relacionales que les resulte más práctico, 
por ejemplo:

 - [H2](http://www.h2database.com/html/main.html) por ser en memoria (especialmente para tests) `libraryDependencies += "com.h2database" % "h2" % "1.4.197" % Test`
 - [SQLite](https://www.sqlite.org/index.html) por ser sencillo `libraryDependencies += "org.xerial" % "sqlite-jdbc" % "3.21.0.1"`
 
## Tablas

Dada la clase de dominio:

```scala
case class Perro(nombre:String, edad:Int)
```

Querríamos poder declarar una tabla equivalente a:

```sql
CREATE TABLE Perros (
    nombre varchar(255),
    edad int 
);
```

Por ejemplo, a modo ilustrativo:

```scala
object Perros extends Table[Perro](
  tableName = "Perros"
) {
  def nombre = column[String]("nombre")
  def edad = column[Int]("edad")
  
  def * = (nombre, edad)
}
```

Donde el método `*` es la proyección de la tabla usada para queries e inserts.
Esta sintaxis puede ir variando a medida que surjan limitaciones en la implementación.

Lo importante es que sea cómodo para el usuario de nuestra librería declarar tablas. 
Es decir, queremos que sea lo más expresiva y declarativa que se pueda.

## Queries

Nos gustaría tratar a las tablas como si fueran colecciones, 
de manera tal que se pueda escribir algo como:


```scala

    val viejitos:Query[String] = query(Perros)
            .filter(_.edad > 12)
            .map(_.nombre)

    val nombres:Seq[String] = viejitos.run()
    
```

Para cumplir con esta sintaxis, será necesario implementar `filter` y `map`, 
así como también el operador `>`. 

Nótese que:

 - La consulta no se ejecuta hasta que no se envíe el mensaje `run`.
 - `_.edad > 12` de alguna forma deberá ser "compilado" a SQL más adelante.
 - A un objeto `Query` debería poder pedirle el código SQL que genera.

### Implementar transformaciones:

 - `filter` compila a un `WHERE`.
 - `map` determina la proyección.
 - `sortBy` determina el orden de los resultados.
 - `take` toma los primeros n elementos del resultset.
 - `drop` saltea los primeros n elementos.
 
 
### Implementar operadores:
 
 Aritméticos:
 
  - `+`	Add	
  - `-`	Subtract	
  - `*`	Multiply	
  - `/`	Divide	
  - `%`	Modulo	
 
 Comparaciones:
 
  - `===` Equal to	
  - `>`	 Greater than	
  - `<`	 Less than	
  - `>=`	 Greater than or equal to	
  - `<=`	 Less than or equal to	
  - `=!=` Not equal to
  
  
### Ejemplos:
 
 ```scala
val perros = query(Perros)

println(perros.drop(2).take(4).sql)
//   select nombre, edad
//     from Perros
//     limit 2 offset 4;

println(perros.filter(_.nombre === "fido").map(_.edad).sql)
//   select edad
//     from Perros
//     where nombre = 'fido';

println(perros.sortBy(_.nombre.desc).sql)
//   select nombre, edad
//     from Perros
//     order by nombre desc;

 ```

### Unions

Dadas dos queries, queremos poder unir sus resultados:

```scala
val perros = query(Perros)

val fidos = perros.filter(_.nombre === "fido")
val cachorros = perros.filter(_.edad < 2)

val fidosYCachorros = fidos ++ cachorros

//   select nombre, edad
//     from Perros
//     where nombre = 'fido'
//   union all select nombre, edad
//     from Perros
//     where edad < 2
```

Nótese que es posible escribir luego algo como:

```scala
val notBoby = fidosYCachorros.filter(_.nombre =!= 'boby').map(_.nombre)

// Donde el SQL generado se vería:
// 
//   select r.nombre
//   from (select nombre, edad
//           from Perros
//           where nombre = 'fido'
//         union all select nombre, edad
//           from Perros
//           where edad < 2
//        ) as r
//   where r.nombre <> 'boby'

```

En este sentido, puede considerarse que `query(Perros)` es un tipo al que podemos llamar
 `TableQuery` y cada subsecuente `union` genera otra `TableQuery`. 
Dado que los selects pueden anidarse, hay que tener en cuenta cómo generar los aliases que 
usemos para las tablas.
 
No nos interesa la eficiencia, con lo cual no nos preocupa usar selects dentro de un from.


## Update, Insert, Delete

Dada una instancia, queremos poder persistirla:

```scala
val fido = Perro(nombre = 'fido', edad = 8)

val perros = query(Perro)

perros.add(fido)

```

O modificar un valor:

```scala

val fido = query(Perro).filter(_name === 'fido')

fido.update(_.edad, 9)

```

Tambien nos gustaría poder persistir objetos que se encuentren en una lista heterogénea:


```scala

val schema = ... // de alguna manera debería tener registradas todas las tablas

// cada objeto es una instancia de una case class diferente
val objects = List(unPerro, unEdificio, unAuto)

// queremos poder agregar cada objeto a su correspondiente tabla
schema.addAll(objects)

```

## Tips:

Las case classes proveen varios métodos útiles:

 - `tupled` retorna una funcion que espera una tupla y devuelve una instancia de la clase.
 - `unapply` retorna una función que espera una instancia de la clase y retorna un Option de una tupla.
 - `copy` permite generar una nueva instancia variando ciertos atributos. Ej.: `fido.copy(edad=10)`.

## Bonus

Implementar foreign keys y permitir joins entre tablas.
