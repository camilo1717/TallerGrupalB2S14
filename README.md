# TallerGrupalB2S14

## INTEGRANTES: 
-John Saritama

-Camilo Fierro

-Luis Sarango

## 1. Genere el archivo CSV para los siguientes datos de estudiantes:

![image](https://github.com/user-attachments/assets/7557f9b1-70f7-4661-94d2-4b1de854c72e)

```Scala
package config
import cats.effect.{IO, Resource}
import com.typesafe.config.ConfigFactory
import doobie.hikari.HikariTransactor
import scala.concurrent.ExecutionContext
object Database {
  private val connectEC: ExecutionContext = ExecutionContext.global
  def transactor: Resource[IO, HikariTransactor[IO]] = {
    val config = ConfigFactory.load().getConfig("db")
    HikariTransactor.newHikariTransactor[
      IO
    ](
      config.getString("driver"),
      config.getString("url"),
      config.getString("user"),
      config.getString("password"),
      connectEC // ExecutionContext requerido para Doobie
    )
  }
}
```

## 2. Genere la tabla en MYSQL para esta información.
![image](https://github.com/user-attachments/assets/43a2b48c-d19f-40bf-b018-ca1c6712f78e)
```Scala
package dao
import cats.effect.IO
import cats.implicits._
import config.Database
import doobie._
import doobie.implicits._
import models.Estudiante
object PersonasDAO {
  // Método para insertar un estudiante
  def insert(estudiante: Estudiante): ConnectionIO[Int] = {
    sql"""
      INSERT INTO estudiantes (nombre, edad, calificacion, genero)
      VALUES (
        ${estudiante.nombre},
        ${estudiante.edad},
        ${estudiante.calificacion},
        ${estudiante.genero}
      )
    """.update.run
  }
  // Método para insertar una lista de estudiantes
  def insertAll(estudiantes: List[Estudiante]): IO[List[Int]] = {
    Database.transactor.use { xa =>
      estudiantes.traverse(e => insert(e).transact(xa))
    }
  }
  // **Nuevo método**: Obtener todos los estudiantes de la base de datos
  def getAll: IO[List[Estudiante]] = {
    val query = sql"""
      SELECT nombre, edad, calificacion, genero
      FROM estudiantes
    """.query[Estudiante].to[List] // Ejecuta la consulta y mapea a una lista de Estudiantes
    Database.transactor.use { xa =>
      query.transact(xa)
    }
  }
}
```


## 3. Elabore un programa que inyecte los datos del archivo CSV a la base de datos. 

```Scala
package models
// Modelo para la tabla "estudiantes"
case class Estudiante(
                       nombre: String,       // Nombre del estudiante
                       edad: Int,            // Edad del estudiante
                       calificacion: Int,    // Calificación del estudiante
                       genero: String        // Género del estudiante ('M' o 'F')
                     )
```
## 4. En el mismo programa agregue la funcionalidad para obtener de la base de datos todos los registros de Estudiantes. 

```Scala
import cats.effect.{IO, IOApp}
import kantan.csv._
import kantan.csv.ops._
import kantan.csv.generic._
import java.io.File
import models.Estudiante
import dao.PersonasDAO
import cats.implicits._ // Esto añade métodos como `traverse` a las colecciones
object Main extends IOApp.Simple {
  val path2DataFile2 = "src/main/resources/data/estudiantes.csv"
  val dataSource = new File(path2DataFile2)
    .readCsv[List, Estudiante](rfc.withHeader.withCellSeparator(','))
  val estudiantes = dataSource.collect {
    case Right(estudiante) => estudiante
  }
  // Secuencia de operaciones IO usando for-comprehension
  def run: IO[Unit] = for {
    // Inserta todos los registros en la base de datos
    insertResult <- PersonasDAO.insertAll(estudiantes)
    _ <- IO.println(s"Registros insertados: ${insertResult.size}")
    // Obtiene todos los registros y los imprime
    allStudents <- PersonasDAO.getAll
    _ <- allStudents.traverse(s => IO.println(s))
  } yield ()
}
```
## 5. Documente en un repositorio de github (archivo README.md) y suba el código en el mismo repositorio. 

## 6. Comparta el enlace del repositorio. 



