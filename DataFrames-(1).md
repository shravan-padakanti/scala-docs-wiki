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

E.g.
```scala
// scala type                                     // sql type
case class Person(name: String, age: Int)         StructType(List(StructField("name", StringType, true)
                                                                  StructField("age", StringType, true)))
```

### Complex Data Types can be combined!

It's possible to arbitrarily nest complex data types! For example

![complex_sql_types_nesting.png](https://github.com/rohitvg/scala-spark-4/blob/master/resources/images/complex_sql_types_nesting.png)

## Accessing Spark SQL Types

**Important**: In order to access _any_ of these data types, basic or complex, you must first import Spark SQL types!

```scala
import org.apache.spark.sql.types._
```

## DataFrames Operations Are MOre Structured

When introduced, the DataFrames APU introduced a no.of relational operations. 

The main difeference between the RDD API and the DataFrames API was that DataFrame APIs accept Spark SQL expressions, instead of arbitrary user-defined function literals like we were used to on RDDs. This allows the optimized to understand the the computation represents, and for example with filter, it can often be used to skip reading unnecessary records.

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
* `dataframe.printSchema()`: prints the schema of the `DatFrame` in tree format.

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