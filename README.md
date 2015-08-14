# ActiveSlick

[![Build Status](https://travis-ci.org/strongtyped/active-slick.svg?branch=develop)](https://travis-ci.org/strongtyped/active-slick)

ActiveSlick is a library that offers CRUD operations for Slick 3.0 projects. The main goal is to provide some basic operations to manage the lifecycle of persisted objects (new/persisted/deleted/stale) and enable the implementation of the Active Record Pattern on Slick mapped case classes.

### Main features
- Basic CRUD and auxiliary methods - add/update/save, delete, findById, count and fetchAll (backed by Reactive Streams).
- Generic Id type. 
- Optimistic locking my means of incremental version.
- **ActiveRecord** trait to enable the Active Record Pattern on mapped case classes via class extensions (pimp-my-library style)

### Project artifact

The artifacts are published to Sonatype Repository. Simply add the following to your build.sbt.

As of version 0.3.0 we don't support Slick 2.0 anymore. The differences between Slick 2.0 and Slick 3.0 are so huge that it makes impossible to support two versions. 

  libraryDependencies += "io.strongtyped" %% "active-slick" % "0.3.0"
  
Source code for version 0.3.0 can be found at:
https://github.com/strongtyped/active-slick/tree/v0.3.0


The version supporting Slick 2.0 is still available on Sonatype Repo. However, this documentation only covers the current version (i.e.: 0.3.0).

  libraryDependencies += "io.strongtyped" %% "active-slick" % "0.2.2"

Source code for version 0.2.2 can be found at:
https://github.com/strongtyped/active-slick/tree/v0.2.2


### Motivation

Slick is able to map result sets to case classes or Tuples because of its isomorphism. This is done thanks to built-in Scala features. However, there is no direct link between case class fields and database columns. Everything is done based on isomorphic projections.

As a consequence, managing of Entities IDs must be done by hand, over and over again. One needs to save a model, ask Slick to return the generated ID and add it explicitly to the case class.

The following code fragment demonstrates how this is typically done in a Slick application.  


```scala
  import slick.driver.H2Driver.api._    
  case class Coffee(name: String, id: Option[Int] = None) // #<1>

  class CoffeeTable(tag: Tag) extends Table[Coffee](tag, "COFFEE") {
    def name = column[String]("NAME")
    def id = column[Int]("ID", O.PrimaryKey, O.AutoInc) // #<2>
    def * = (name, id.?) <>(Coffee.tupled, Coffee.unapply)
  }

  val Coffees = TableQuery[CoffeeTable]

  val coffee = Coffee("Colombia")
  val insertAction = Coffees.returning(Coffees.map(_.id)) += coffee // #<3>
```

Both **Coffee** and **CoffeeTable** have an **Id** representing the primary key. **Coffee** has a field of type **Option[Int]** (#1) and **CoffeeTable** has a method returning a **Rep[Int]** (#2). However, in order to inserta new Coffee we need some boilerplate to get back the generated ID (#3).

If we manage to connect both **ID** we can provide some generic functionality to manage Entities. Inserting a new Entity is only one of the possible use cases. With a well known ID, we can distinguish if an Entity has been already persisted or not, we can easilly implement byId methods (deleteById, findById) and we can add extensions methods like: save, udpate, delete


### History
The first intention of this project was to implement Slick DAOs or Repositories using only Scala features and the Scala compiler (no code generation).

In the time of Slick 2.0, this library required the usage of the Cake Pattern to wire together all the parts wihtout pre-defining a driver.

Since Slick 3.0 imposes a huge refactoring, we decided to turn ActiveSlick inside out and eliminate the need of the Cake Pattern.


### Mapping using ActiveSlick

The following code fragment illustrates how we can use ActiveSlick **EntityActions** to bind the model Id field and the table id column.  

```scala
    object MappingWithActiveSlick {
    
      case class Coffee(name: String, id: Option[Int] = None) extends Identifiable {
        override type Id = Int
      }
    
      object CoffeeRepo extends EntityActions(H2Driver) {
    
        import jdbcProfile.api._
    
        class CoffeeTable(tag: Tag) extends Table[Coffee](tag, "COFFEE") {
          def name = column[String]("NAME")
          def id = column[Int]("ID", O.PrimaryKey, O.AutoInc)
          def * = (name, id.?) <>(Coffee.tupled, Coffee.unapply)
        }
    
        implicit val baseTypedType: BaseTypedType[Id] = implicitly[BaseTypedType[Id]] // #<1>
        type Entity = Coffee // #<2>
        type EntityTable = CoffeeTable // # <3>
        val tableQuery = TableQuery[CoffeeTable] // # <4>
    
        def $id(table: CoffeeTable) = table.id // # <5>
        val idLens = lens[Coffee, Option[Int]]( // # <6>
          coffee => coffee.id,
          (coffee, id) => coffee.copy(id = id)
        )
      }
    
      implicit class EntryExtensions(val entity: Coffee) extends ActiveRecord[Coffee] {
        val crudActions = CoffeeRepo
      }
    
      val saveAction = Coffee("Colombia").save()
    }
```

The mapping is basically the same. Except that Coffee implements **Identifiable** However, we define it inside **CoffeeRepo** which implements **EntityActions**.

From #1 to #6 we have the necessary glue code to connect the IDs.

 * (#1) we need to bring to scope a implicit BaseTypedType for the Id type of the `Entity`. Without that ActiveSlick can't be `Queries`.
 * (#2) The type of the `Entity`. We choose to use **Type Aliases** instead of **Type Parameters** because it make the code a little bit cleaner when composing with other triats. (see `OptimisticLocking` trait and `SupplierDao` in `Schema.scala`).
 * (#3) **Type Alias** for the Table type
 * (#4) The `TableQuery` as usual.
 * (#5) A mapping method that allow us to identity the Slick Rep (column) tha is pointing to the Entity id column.
 * (#6) A lens for the `Coffee` id so we can extract the id and copy it to the case class.

We have now a well known column for our primary key. The next step is to define the method to extract the id from our model and to add a generated id back into the model. 

### Transactions
As of version 0.3.0 and the introduction of `DBIO` in Slick 3.0, all methods return `DBIO` (not `Futures`). In Slick 3.0 the DB sessions and the transactional sessions are not passed as implicit parameters therefore is the user that have to manage the sessions and transactions.

If ActiveSlick were returning Futures instead, then the transactions will have to be managed internally by **ActiveSlick** which is of course not desirable.

### Optimistic Locking
TODO
(see `OptimisticLocking` trait and `SupplierDao` in `Schema.scala`).

### Before Insert and Update hooks
TODO
(see `EntityActionsBeforeInsertUpdateTest.scala`)

