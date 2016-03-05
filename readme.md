# Scala Query AST

The goal of this AST is to sufficiently generalize a querying AST such that it can be used for many platforms.

As it stands now, there is a quick 'n dirty SQL DSL, and a first pass implementation of a simple, high level AST.

The sample below uses the SQL DSL, and is a valid query.

```scala
import com.github.jacoby6000.query.ast._
import com.github.jacoby6000.query.interpreter
import com.github.jacoby6000.query.dsl.sql._

val q =
  select (
    p"foo" ++ 10 as "woozle",
    `*`
  ) from "bar" leftOuterJoin "baz" on (
    p"bar.id" === p"baz.barId"
  ) innerJoin "biz" on (
    p"biz.id" === p"bar.bizId"
  ) where (
    p"biz.name" === "LightSaber" and
    p"biz.age" > 27
  ) orderBy p"biz.age".desc groupBy p"baz.worth".asc

interpreter.interpretSql(q.query)
```

The sql output of this would be

```sql
SELECT
    foo + 10 AS woozle,
    *
FROM
    bar
LEFT OUTER JOIN
    baz
        ON bar.id = baz.barId
INNER JOIN
    biz
        ON biz.id = bar.bizId
WHERE
    biz.name = “LightSaber”
    AND  biz.age > 27
ORDER BY
    biz.age DESC
GROUP BY
    baz.worth ASC
```

As a proof of concept, here are some examples translated over from the book of doobie

```scala
import com.github.jacoby6000.query.ast._
import com.github.jacoby6000.query.interpreter
import com.github.jacoby6000.query.dsl.sql._
import com.github.jacoby6000.query.dsl.doobie._
import doobie.imports._
import shapeless.HNil

import scalaz.concurrent.Task

case class Country(code: String, name: String, pop: Int, gnp: Option[Double])

val xa = DriverManagerTransactor[Task](
  "org.postgresql.Driver", "jdbc:postgresql:world", "postgres", ""
)

val baseQuery =
  select("code", "name", "population", "gnp") from p"country"

def biggerThan(n: Int) =
  (baseQuery where p"population" > `?`)
    .query[Country]
    .prepare(n)
    .list

val biggerThanRun = biggerThan(150000000).quick.run
    /*List(
        Country(BRA,Brazil,170115000,Some(776739.0))
        Country(IDN,Indonesia,212107000,Some(84982.0))
        Country(IND,India,1013662000,Some(447114.0))
        Country(CHN,China,1277558000,Some(982268.0))
        Country(PAK,Pakistan,156483000,Some(61289.0))
        Country(USA,United States,278357000,Some(8510700.0))
      )*/

def populationIn(r: Range) =
  (baseQuery where (
    p"population" >= `?` and
    p"population" <= `?`
  )).query[Country]
    .prepare((r.min, r.max))
    .list

val populationInRun = populationIn(1000 to 10000).transact(xa).run
    /*List(
        Country(BRA,Brazil,170115000,Some(776739.0)),
        Country(PAK,Pakistan,156483000,Some(61289.0))
      )*/
```