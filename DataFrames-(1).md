So far we saw what `DataFrame`s are, how to create them and how to use SQL queries on them.

**`DataFrame`s have their own APIs as well!**

## DataFrames Data Types

To enable optimization, Spark SQL's `DataFrame`s operate on a **restricted set of data types**.

> Thus, since we have to provide some kind of shchema to Spark when we create a DataFrame, the types that are used by Spark are Spark SQL Data Types i.e. the ones below corresponding to the types we provide in schema. If we have a datatype that does not have a corresponding Spark SQL Type, then we cannot use Spark SQL with that type.

Basic Spark SQL Data Types:
```
Scala Type                 SQL Type            Details
---------------------------------------------------------------------------------------------------------
Byte                       ByteType            1 byte signed integers (-128, 127)
Short                      ShortType           2 byte signed integers (-32768, 32767)
Int                        IntegerType         4 byte signed integers (-2147483648, 2147483647)
Long                       LongType            8 byte signed integers
java.math.BigDecimal       DecimalType         Arbitrary precision signed decimals
Float                      FloatType           4 byte floating point number 
Double                     DoubleType          8 byte floating point number     
Array[Byte]                BinaryType          Byte sequence values        
Boolean                    BooleanType         true/false       
Boolean                    BooleanType         true/false       
java.sql.Timestamp         TimestampType       Date containing year, month, day, hour, minute, second
java.sql.Date              DateType            Date containing year, month, day          
String                     StringType          Character string values (stored as UTF8)     
```

Complex Spark SQL Data Types:
```
Scala Type                 SQL Type
-------------------------------------------------------------------
Array[T]                   ArrayType(elementType, containsNull)
Map[K, V]                  MapType(keyType, valueType, valueContainsNull)
case class                 StructType(List[StructFields])
```

### Arrays

Array of only one type of element (`elementType`). 
`containsNull` is set to `true` if the elements in `ArrayType` value can have null values

E.g.
```scala
// scala type           // sql type
Array[String]            ArrayType(StringType, true)
```

### Maps

Map of key/value pairs with two type of elements.
`valuecontainsNull` is set to `true` if the elements in `MapType` value can have null values.

E.g.
```scala
// scala type           // sql type
Map[Int, String]        Map(IntegerType, StringType, true)
```
### Structs

Struct type with the list of possible fields of different types.
`containsNull` is set to `true` if the elements in `StructType` can have null values.

`Class` in Scala is realized using a `StructType`, and `ClassVariables` using `StructFields`.
E.g.
```scala
// scala type                                     // sql type
case class Person(name: String, age: Int)         StructType(List(StructField("name", StringType, true)
                                                                  StructField("age", StringType, true)))
```

### Complex Data Types can be combined!

It's possible to arbitrarily nest complex data types! For example, below the `Project` type is defined in Scala on the left, and in Spark SQL type on the right.

![complex_sql_types_nesting.png](https://github.com/rohitvg/scala-spark-4/blob/master/resources/images/complex_sql_types_nesting.png)

## Accessing Spark SQL Types

**Important**: In order to access _any_ of these data types, basic or complex, you must first import Spark SQL types!

```scala
import org.apache.spark.sql.types._
```

## DataFrames Operations Are More Structured

When introduced, the DataFrames API introduced a no. of relational operations. 

The main difference between the RDD-API and the DataFrames-API was that DataFrame APIs accept Spark SQL expressions, instead of arbitrary user-defined function literals like we were used to on RDDs. This allows the optimized to understand the the computation represents, and for example with filter, it can often be used to skip reading unnecessary records.

## DataFrames API

Similar looking to SQL: Example methods include:

* `select`
* `where`
* `limit`
* `orderBy`
* `gorupBy`
* `join`

## Getting a look at your data

Before we get into transformations and actions on `DataFrame`s, lets first look at the ways we can have a look at our dataset.

* `dataframe.show()`: pretty-prints `DatFrame` in tabular form. Shows first 20 elements.

   ```scala
    case class Employee(id: Int, fname: String, lname: String, age: Int, city: String)
    
    // DataFrame with schema defined in Employee case class
    val employeeDF = sc.parallelize(...).toDF
    employeeDF.show()
    
    // employeeDF:
    // +---+-----+-------+---+--------+
    // | id|fname| lname |age| city   |
    // +---+-----+-------+---+--------+
    // | 12|  Joe|  Smith| 38|New York|
    // |563|Sally|  Owens| 48|New York|
    // |645|Slate|Markham| 28|  Sydney|
    // |221|David| Walker| 21|  Sydney|
    // +---+-----+-------+---+--------+
   ```

* `dataframe.printSchema()`: prints the schema of the `DatFrame` in tree format.

   ```scala
    case class Employee(id: Int, fname: String, lname: String, age: Int, city: String)
    
    // DataFrame with schema defined in Employee case class
    val employeeDF = sc.parallelize(...).toDF
    employeeDF.printSchema()
    
    // root
    //  |-- id: integer (nullable = true)
    //  |-- fname: string (nullable = true)
    //  |-- lname: string (nullable = true)
    //  |-- age: integer (nullable = true)
    //  |-- city: string (nullable = true)

   ```


## Common DataFrame transformations

Like on RDDs, transformation on DataFrames are:

1. operations which return a `DataFrame` as a results
2. lazily evaluated.

Some common transformations include:

```scala
def select(col: String, cols: String*): DataFrame
// selects a set of named columns anreturns a new DataFrame with these columns as a result

def agg(expr: Column, expr: Column*): DataFrame
// performs aggregations on a series of columns and returns a new DataFrame with the calculated output

def groupBy(col1: String, cols: String*): DataFrame //simplified
// groups the DataFrame using the specified columns. Intended to used before an aggregation.

def join(right: DataFrame): DataFrame //simplified
// inner join with another DataFrame
```

Other transformations include: `filter`, `limit`, `orderBy`, `where`, `as`, `sort`, `union`, `drop`, amongst others.

### Specifying Columns

As seen above, most methods take a parameter of type `Column` or `String`, thus always referring to a column/attribute in the dataset.

**Most methods on `DataFrame`s tend to work with some well defined operation on column of the data set.**

There are 3 ways:

1. Using the **$** notation: 
    ```scala
    // requires "import spark.implicits._
    df.filter($"age" > 18)
    ```
1. Referring to a `DataFrame`: 
    ```scala
    df.filter(df("age") > 18)
    ```
1. Using **SQL query string**: 
    ```scala
    df.filter("age > 18")  // sometimes is error prone. So use the above 2.
    ```

## DataFrame Transformations: Example

[Recall the previous example](https://github.com/rohitvg/scala-spark-4/wiki/Spark-SQL#an-interesting-sql-query) we saw. **How do we solve this using the DataFrame-API?**

```scala
case class Employee(id: Int, fname: String, lname: String, age: Int, city: String)

// DataFrame with schema defined in Employee case class
val employeeDF = sc.parallelize(...).toDF

// No need to register here like we did previously

// employeeDF:
// +---+-----+-------+---+--------+
// | id|fname| lname |age| city   |
// +---+-----+-------+---+--------+
// | 12|  Joe|  Smith| 38|New York|
// |563|Sally|  Owens| 48|New York|
// |645|Slate|Markham| 28|  Sydney|
// |221|David| Walker| 21|  Sydney|
// +---+-----+-------+---+--------+

val sydneyEmployeesDF = sparkSession.select("id", "lname")
                                    .where("city = sydney")
                                    .orderBy("id")

// sydneyEmployeesDF:
// +---+-------+
// | id|  lname|
// +---+-------+
// |221| Walker|
// |645|Markham|
// +---+-------+
```

## Filtering in Spark SQL:

The DataFrame API gives us 2 methods for filtering: `filter` and `where`. **They are equivalent!**

```scala
val over30 = employeDF.filter("age > 30").show()
// same as 
val over30 = employeDF.where("age > 30").show()
```

Filters can be complex as well, using logical operators, groups, etc:

```scala
employeDF.filter(($"age" > 30) && ($"city" === "sydney")).show()
```

## Grouping and aggregating on DataFrames

One of the most common tasks on tables is to: (1) group data by a certain attribute, and then
(2) do some kind of aggregation on it, like count.

For grouping and aggregating, Spark SQL provides:

* a `groupBy` function which returns a `RelationalGroupedDataSet`
* The `RelationalGroupedDataSet` has several standard aggregation functions defined on it like `count`, `sum`, `max`,`min`,`avg`,`agg`. 

So, how to group and aggregate:

1. Call `groupBy` on a column/attribute of a DataFrame.
2. On the resulting `RelationalGroupedDataSet`, call one of `count`, `max`, or `agg`. Here for `agg` also specify which column/attribute to call the subsequent functions upon.

```scala
df.groupBy($"attribute1")
  .agg(sum($"attribute2"))

df.groupBy($"attribute1")
  .count($"attribute2")
```

### Example

We have a dataset of homes available for sale. Lets calculate the most and least expensive home per zip code.

```scala
case class listings(street: String, zip: Int, price: Int)
val listingDF = ...

import org.apache.spark.sql.functions._

val mostExpensiveDF = listings.groupBy($"zip")
                              .max($"price")

val leastExpensiveDF = listings.groupBy($"zip")
                              .min($"price")
```

We have datasets of all the posts in an online forum. We want to tally up each authors posts per subforum, and then rank he authors with the most posts per subforum
```scala
case class post(authorId: Int, subForum: String, likes: Int, date: String)
val postsDF = ...

import org.apache.spark.sql.functions._

val rankedDF = post.groupBy($"authorId", $"subForum")
                   .agg(count($"authorId")) // new DF with columns: authorId,, subForum, count(authorId) 
                   .orderBy($"subForum", $"count(authorId)".desc)

// postsDF:
// +---------+--------+-------+-------+
// | authorId|subForum| likes |dates  |
// +---------+--------+-------+-------+
// |        1|  design|      2| "2012"|
// |        1|  debate|      0| "2012"|
// |        2|  debate|      0| "2012"|
// |        3|  debate|     23| "2012"|
// |        1|  design|      1| "2012"|
// |        1|  design|      0| "2012"|
// |        2|  design|      0| "2012"|
// |        2|  debate|      0| "2012"|
// +---------+--------+-------+-------+

// rankedDF:
// +---------+--------+---------------+
// | authorId|subForum|count(authorId)|
// +---------+--------+---------------+
// |        2|  debate|              2|
// |        1|  debate|              1|
// |        3|  debate|              1|
// |        1|  design|              3|
// |        2|  design|              1|
// +---------+--------+-------+-------+
```

Finally: 

* `RelationalGroupedDataset` API: https://spark.apache.org/docs/latest/api/scala/index.html#org.apache.spark.sql.RelationalGroupedDataset
* Methods within `agg`: https://spark.apache.org/docs/latest/api/scala/index.html#org.apache.spark.sql.functions$