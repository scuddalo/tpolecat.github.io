---
layout: book
number: 7
title: DDL, Inserting, and Updating
---


### Setting Up

Again we set up an H2 transactor and pull in YOLO mode, but this time we're not using the world database.

```scala
import doobie.imports._

import scalaz._, Scalaz._, scalaz.concurrent.Task

val xa = DriverManagerTransactor[Task](
  "org.h2.Driver",                      
  "jdbc:h2:mem:ch7;DB_CLOSE_DELAY=-1",  
  "sa", ""                              
)

import xa.yolo._
```

### Data Definition

Let's create a new table, which we will use for the examples to follow. This looks a lot like our prior usage of the `sql` interpolator, but this time we're using `update` rather than `query`. The types are indicated.

```scala
scala> (sql"""
     |   CREATE TABLE person (
     |     id   BIGINT IDENTITY,
     |     name VARCHAR(255) NOT NULL UNIQUE,
     |     age  TINYINT
     |   )""".update        // Update0
     |       .run           // ConnectionIO[Int]
     |       .transact(xa)  // Task[Int]
     |       .run)          // Int
res0: Int = 0
```



### Inserting


Inserting is straightforward and works just as with selects.

```scala
scala> def insert1(name: String, age: Option[Int]): Update0 =
     |   sql"insert into person (name, age) values ($name, $age)".update
insert1: (name: String, age: Option[Int])doobie.imports.Update0

scala> insert1("Alice", Some(12)).quick.run
1 row(s) updated

scala> insert1("Bob", None).quick.run
1 row(s) updated
```

And read them back.

```scala
case class Person(id: Long, name: String, age: Option[Int])
```

```scala
scala> sql"select id, name, age from person".query[Person].quick.run
Person(1,Alice,Some(12))
Person(2,Bob,None)
```


### Updating


Updating follows the same pattern.

```scala
scala> sql"update person set age = 15 where name = 'Alice'".update.quick.run
1 row(s) updated

scala> sql"select id, name, age from person".query[Person].quick.run
Person(1,Alice,Some(15))
Person(2,Bob,None)
```

### Retrieving Results

Of course when we insert we usually want the row back, so let's do that. First we'll do it the hard way, by inserting, getting the last used key via H2's `identity()` function, then selecting the indicated row. 

```scala
def insert2(name: String, age: Option[Int]): ConnectionIO[Person] =
  for {
    _  <- sql"insert into person (name, age) values ($name, $age)".update.run
    id <- sql"call identity()".query[Int].unique
    p  <- sql"select id, name, age from person where id = $id".query[Person].unique
  } yield p
```

```scala
scala> insert2("Jimmy", Some(42)).quick.run
Person(3,Jimmy,Some(42))
```

This is irritating but it is supported by all databases (although the "get the last used id" function will vary by vendor). A nicer way to do this is in one shot by returning specified columns from the inserted row. H2 supports this feature.


```scala
def insert3(name: String, age: Option[Int]): ConnectionIO[Person] = {
  sql"insert into person (name, age) values ($name, $age)"
    .update
    .withUniqueGeneratedKeys[Person]("id", "name", "age") // ConnectionIO[Person]
}
```

```scala
scala> insert2("Elvis", None).quick.run
Person(4,Elvis,None)
```

This mechanism also works for updates, for databases that support it (PostgreSQL for example, but not H2). In the case of multiple row updates use `.withGeneratedKeys[A](cols...)` to get a `Process[ConnectionIO, A]`.


