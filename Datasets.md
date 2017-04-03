When we call a `collect()` on DataFrames,  we get an `Array[org.apaceh.spark.sql.Row]`.

Going back to the [listings example](https://github.com/rohitvg/scala-spark-4/wiki/DataFrames-(1)#example):

```scala
case class Listing(street: String, zip: Int, price: Int)
val list = List(Listing("Camino Verde Dr", 95119, 50000),
                Listing("Burnett St", 12345, 20000),
                Listing("Lawerence expy", 12345, 25000),
                Listing("El Camino", 95119, 30000))
val pricesDF = SparkHelper.sc.parallelize(list).toDF
pricesDF.show()

// +---------------+-----+-----+
// |         street|  zip|price|
// +---------------+-----+-----+
// |Camino Verde Dr|95119|50000|
// |     Burnett St|12345|20000|
// | Lawerence expy|12345|25000|
// |      El Camino|95119|30000|
// +---------------+-----+-----+

import org.apache.spark.sql.functions._

val averagePricesDF = pricesDF.groupBy($"zip")
                              .avg($"price")
averagePricesDF.show()

// +-----+----------+
// |  zip|avg(price)|
// +-----+----------+
// |95119|   40000.0|
// |12345|   22500.0|
// +-----+----------+

val averagePrices = averagePricesDF.collect()
// averagePrices : Array[org.apaceh.spark.sql.Row]
// [95119,40000.0]
// [12345,22500.0]
```
averagePrices is of type `Array[org.apaceh.spark.sql.Row]`, and not an array of Doubles as expected.

What is this?

Since DataFrames are made up of `Row`s , and don't have any type information, we have to cast things:
Lets pass `String` and `Int` and see if it works:
```scala
val averagePricesAgain = averagePrices.map {
                    row => ( row(0).asInstanceOf[String], row(1).asInstanceOf[Int] )
}
```

**But** this gives `java.lang.ClassCastException`!

Let's try to see that's in this Row.

```scala
averagePrices.head.schema.printTreeString()
// root
// |--zip: integer (nullable = true)
// |--avg(price): double (nullable = true)

```

Based on this we update our cast, and it works:

```scala
val averagePricesAgain = averagePrices.map {
                    row => ( row(0).asInstanceOf[Int], row(1).asInstanceOf[Double] ) // Ew - hard to read!
}
// averagePricesAgain : Array[(Int, Double)]
```

**Wouldn't it be nice if we could have both: Spark SQL Optimizations and typesafety?** so that after we call collect then we could simply use a `.price` or something to get the avg value?

DataFrames are **untyped**, so they can't help here. This is where **Datasets** come into play.

## Datasets

**DataFrames are actually DataSets**

```scala
type DataFrame = Dataset[Row]
```

What the heck is a Dataset?

* **Typed** distributed collections of data.
* unify the `DataFrame` and the `RDD` APIs. **Mix and Match!**
* require structured/semi-structured data. Schemas and Encoders are core part of Datasets (just like DataFrames).

**Things of Datasets as a compromise between RDDs and DataFrames.** You get more type information on Datasets than on DatFrames, and you get more optimizations than you get on RDDs.

> Dataset API is an extension to DataFrames that provides a type-safe, object-oriented programming interface. It is a strongly-typed, immutable collection of objects that are mapped to a relational schema.
>
> At the core of the Dataset, API is a new concept called an encoder, which is responsible for converting between JVM objects and tabular representation. The tabular representation is stored using Spark internal Tungsten binary format, allowing for operations on serialized data and improved memory utilization. Spark 1.6 comes with support for automatically generating encoders for a wide variety of types, including primitive types (e.g. String, Integer, Long), Scala case classes, and Java Beans.

**Example**

Going back to the [listings example](https://github.com/rohitvg/scala-spark-4/wiki/DataFrames-(1)#example):

Lets calculate the average home price per zipcode with Datasets this time. Assuming listingDS is of type `Dataset[Listing]` instead of a DataFrame:

```scala
val pricesDS = SparkHelper.sc.parallelize(list).toDS
pricesDS.show()

// +---------------+-----+-----+
// |         street|  zip|price|
// +---------------+-----+-----+
// |Camino Verde Dr|95119|50000|
// |     Burnett St|12345|20000|
// | Lawerence expy|12345|25000|
// |      El Camino|95119|30000|
// +---------------+-----+-----+

val averagePricesDS = pricesDS.groupByKey( row => row.zip )    // looks like groupByKey on RDDs!
                              .agg( avg($"price").as[Double] ) // looks like our DataFrame operators! Here we are telling spark that the average price is of type Double.
averagePricesDS.show()

// +-----+----------+
// |value|avg(price)|
// +-----+----------+
// |95119|   40000.0|
// |12345|   22500.0|
// +-----+----------+
```

We can freely mix APIs!

So, **Datasets are something in the middle between DataFrames and RDDs**. 

* We can still use relational `DataFrame` operations that we learned on `DataSet`s
* `DataSet`s add more typed operations that can be used as well
* `DataSet`s let you use higher-order functions like `map`, `flatMap`, `filter` again! We cannot use these on plain DataFrames.

So Datasets can be used when we want a mix of functional and relational transformation while benefitting from some of the optimizations on DataFrames.

And we've **almost** got a type safe API as well which gives us compile time safety.

### Creating DataSets

#### From a DataFrame

We use the `toDS` convenience method

```scala
val conf: SparkConf = new SparkConf
val sc: SparkContext = new SparkContext(master = "local[*]", appName = "foo", conf)
val sqlContext = new SQLContext(sc)
import sqlContext.implicits._  //required

myDF.toDS
```

Note that it is often desirable to read in data from JSON file, which can be done with the read method on the `SparkSession` object like we saw in previous sessions, and then converted to a Dataset:

```scala
// we need to specify the type information as Datasets are typed!
val myDS = sparkSession.read.json("people.json"").as[Person] 
```

#### From an RDD

We use the `toDS` convenience method

```scala
val conf: SparkConf = new SparkConf
val sc: SparkContext = new SparkContext(master = "local[*]", appName = "foo", conf)
val sqlContext = new SQLContext(sc)
import sqlContext.implicits._  //required

myRDD.toDS
```

#### From an RDD
We use the `toDS` convenience method

```scala
val conf: SparkConf = new SparkConf
val sc: SparkContext = new SparkContext(master = "local[*]", appName = "foo", conf)
val sqlContext = new SQLContext(sc)
import sqlContext.implicits._  //required

List("yay", "ohnoes", "hurray").toDS
```

## Typed Columns

Recall the `Column` type from `DataFrame`s. In `Dataset`s, typed operations tend to act on `TypedColumn` instead.

```
<console>: 58: error: type mismatch;
found    : org.apache.spark.sql.Column
required : org.apache.spark.sql.TypedColumn[...]
                  .agg(avg($"price")).show
```

To create a `TypedColumn`, all we have to do is call `as[..]` on our untyped Column. i.e.
```scala
$"price".as[Double]
```

## Transformations on Datasets:

The `Dataset` API includes both untyped and typed transformations:

* **Untyped transformations**: the ones we learned on `DataFrame`s.
* **Typed transformations**: typed variants of `DataFrame` transformations **+** additional transformations such as RDD-like higher order functions `map`, `flatMap`, etc.

Since `DataFrames` are nothing but `DataSets`, **the APIs are integrated**. For example, we can call a `map` on a `DataFrame`, and get back a `Dataset`. Though, since we are using a Dataset function on a DataFrame, we have to **explicitly provide type information** when we go from `DataFrame` to `Dataset` via typed transformation.
> Note that not every operation from RDDs are available on Datasets, and not all of these operations that are available on Datasets  look 100% the same on Datasets as they did on RDDs.

```scala
val keyValuesDF: DataFrame = List( (3, "Me"),(1, "Thi"),(2, "Se"),(3, "ssa"),(3, "-)"),(2, "Cre"),(2, "t") ).toDF
val res: Dataset[Int] = keyValuesDF.map( row => row(0).asInstanceOf[Int] + 1) // Ew!
```

### Common (Typed) Transformation on Datasets

```scala
def map[U](func: T => U): Dataset[U]
// Apply function to each element in the Dataset and return a Dataset of the result.

def flatMap[U](func: T => TraversableOnce[U]): Dataset[U]
// Apply function to each element in the Dataset and return a Dataset of the contents of the iterators returned.

def filter(pred: T => Boolean): Dataset[U]
// Apply predicate function to each element in the Dataset and return a Datset of elements that have passed the predicate condition i.e. pred.

def distinct(): Dataset[T]
// Return Dataset with duplicates removed.

def groupByKey[K](func: T=> K): KeyValueGroupedDataset[K,T]
// Apply function to each element in the Dataset and return a KeyValueGroupedDataset where the data is grouped by the given key func.
// Doesn't return a dataset directly, as it is used with an aggregator operation that we will see below which ultimately returns a Dataset.

def coalesce(numPartitions: Int): Dataset[T]
// Returns a new Dataset that has exactly numPartitions partitions.

def repartition(numPartitions: Int): Dataset[T]
// Returns a new Dataset that has exactly numPartitions partitions.
```

Full API: https://spark.apache.org/docs/latest/api/scala/index.html#org.apache.spark.sql.Dataset

### Grouped Operations on Datasets

Like on `DataFrame`s, `Datasets` have special set of aggregation operations meant to be used after a call to `groupByKey` on a `Dataset`.

* calling `groupByKey` on a `Dataset` returns a `KeyValueGroupedDataset`
* `KeyValueGroupedDataset` contains a no. of aggregation operations which return Datasets.

So, how to group and aggregate:

1. call `groupByKey` on a `Dataset` and get back a `KeyValueGroupedDataset`
2. use an aggregation operation on `KeyValueGroupedDataset` and get back a `Dataset`.


[Compare this with DataFrames](https://github.com/rohitvg/scala-spark-4/wiki/DataFrames-(1)#grouping-and-aggregating-on-dataframes).

**NOTE:** using a `groupBy` on a Dataset gives back a `RelationalGroupedDataset` which after running the aggregation operation returns a `DataFrame`.

#### Some `KeyValueGroupedDataset` Aggregation operations

```scala

def reduceGroups(f: (V, V) ⇒ V): Dataset[(K, V)]
// Reduces the elements of each group (obtained after groupByKey) of data using the specified binary function. The given function must be commutative and associative or the result may be non-deterministic.

def
agg[U](col: TypedColumn[V, U]): Dataset[(K, U)]
// Computes the given aggregation, returning a Dataset of tuples for each unique key and the result of computing this aggregation over all elements in the group.
```

#### Using the General `agg` Operation

Just like on `DataFrames`, there exists a general aggregation operation `agg` defined on `KeyValueGroupedDataset`.

```scala
def agg[U](col: TypedColumn[V, U]): Dataset[(K, U)]
```

Typically, we would pass an operation from function e.g. `avg` with a column to be computed on:

```scala
kvds.agg(avg($"colname"))

// ERROR:
// <console>: 58: error: type mismatch;
// found    : org.apache.spark.sql.Column
// required : org.apache.spark.sql.TypedColumn[...]
                  .agg(avg($"price")).show
```
But this gives the above error as seen. As noted in the definition, we have to pass a `TypedColumn` and not just a column:

```scala
kvds.agg(avg($"colname").as[Double])  // all good!
```

#### Some `KeyValueGroupedDataset` Aggregation operations

```scala
def mapGroups[U](f: (K, Iterator[V]) ⇒ U)(implicit arg0: Encoder[U]): Dataset[U]
// Applies the given function to each group of data. For each unique group, the function will be passed the group key and an iterator that contains all of the elements in the group. The function can return an element of arbitrary type which will be returned as a new Dataset.

def flatMapGroups[U](f: (K, Iterator[V]) ⇒ TraversableOnce[U])(implicit arg0: Encoder[U]): Dataset[U]
// Applies the given function to each group of data. For each unique group, the function will be passed the group key and an iterator that contains all of the elements in the group. The function can return an element of arbitrary type which will be returned as a new Dataset.
```

Full API : https://spark.apache.org/docs/latest/api/scala/index.html#org.apache.spark.sql.KeyValueGroupedDataset

> Note that as of today (April 2017), the `KeyValueGroupedDataset` is marked as `@Expreimental` and `@Evolving` - so it is subject to fluctuations.

## `reduceByKey` in Dataset API?

Look at the `Dataset` API docs, it is missing an important transformation that we often used on RDDs: `reduceByKey`.

### Challenge 

Emulate the semantics of `reduceByKey` on a `Dataset` using `Dataset` operations presented so far. Assume we'd have the following data set:

```scala
val keyValues = List( (3, "Me"),(1, "Thi"),(2, "Se"),(3, "ssa"),(1, "sIsA"),(3, "ge:"),(3, "-)"),(2, "cre"),(2, "t") )
```

Find a way to use the `Datasets` API to achieve the same result as calling `reduceByKey` on an RDD with the same data i.e. 

```scala
keyValuesRDD.reduceByKey(_ + _)
// (2,Secret)
// (3,Message:-))
// (1,ThisIsA)
```

So we do this:

```scala
val keyValuesDS = keyValues.toDS
val keyValGropedDataset = keyValuesDS.groupByKey(row => row._1) //keys are ints, vals are pairs
val simulatedReduced = keyValGropedDataset.mapGroups( (key ,valpair) => (key, valpair.foldLeft("")( (acc, valpair) => acc + valpair._2 )) ).show()

// +---+----------+
// | _1|        _2|
// +---+----------+
// |  1|   ThisIsA|
// |  3|Message:-)|
// |  2|    Secret|
// +---+----------+

simulatedReduced.sort($"_1").show()

// +---+----------+
// | _1|        _2|
// +---+----------+
// |  1|   ThisIsA|
// |  2|    Secret|
// |  3|Message:-)|
// +---+----------+

```

The only issue with this approaci is the disclaimer in the API docs for `mapGroups`:
>This function does not support partial aggregation, and as a result requires **shuffling** all the data in the Dataset. If an application intends to perform an aggregation over each key, it is best to use the reduce function or an org.apache.spark.sql.expressions#Aggregator.

**Thus don't use the `mapGroups` function on `KeyValueGropuedDataset` unless we have to!**

So the docs suggest to use (1) reduce function OR (2) Aggregator

### `reduceByKey` using a reduce function:
```scala
val keyValuesDS = keyValues.toDS
val keyValGropedDataset = keyValuesDS.groupByKey(row => row._1) //keys are ints, vals are pairs
                                     .mapValues(row => row._2)
                                     .reduceGroups( (acc, str) => acc + str )

```

**This works! But docs also suggested an `Aggregator`:

### `reduceByKey` using an Aggregator:

Sometimes the `agg` function is too specialized and we need some custom aggregator. For this we have the **abstract class** **org.apache.spark.sql.expressions.Aggregator**. 

This **abstract class** helps us generically aggregate data (similar to the `aggregate` method we saw on RDDs) by defining some abstract methods. (https://spark.apache.org/docs/latest/api/scala/index.html#org.apache.spark.sql.expressions.Aggregator)

```scala
Class Aggregator[-IN, BUF, OUT]           // org.apache.spark.sql.expressions.Aggregator

// It has 3 types:
// IN: The input type for the aggregation. When we use aggregator after `groupByKey`, this is the type that represents the value in the key/value pair of the KeyValueGroupedDataset.
// BUF: The type of the intermediate value of the reduction.
// OUT: The type of the final output result.
```

This is how we implement our own Aggregator:

```scala
val myAgg = new Aggregator[IN, BUF, OUT] {
    def zero: BUF  = ...                   // Initial value
    def reduce(b: BUF, a: IN): BUF  = ...  // Add an element to the running total
    def merge(b1: BUF, b2: IN): BUF  = ... // Merge Intermediate values
    def finish(b: BUF): OUT  = ...         // Return final result
}.toColumn
``` 

Going back to the example

```scala
val keyValues = List( (3, "Me"),(1, "Thi"),(2, "Se"),(3, "ssa"),(1, "sIsA"),(3, "ge:"),(3, "-)"),(2, "cre"),(2, "t") )
val keyValuesDS = keyValues.toDS

// IN type is that comes out of the "groupByKey" method. 
// We know that our OUT is going to be of type "String"
// So as a logical choice, BUF can be a String
val myAgg = new Aggregator[(Int, String), String, String] {
    def zero: String  = ""                                       // Initial value - empty string
    def reduce(b: String, a: (Int, String)): String  = b + a._2  // Add an element to the running total
    def merge(b1: String, b2: String): String  = b1 + b2         // Merge Intermediate values
    def finish(r: String): String  = r                           // Return final result
}.toColumn

keyValuesDS.groupByKey( pair => pair._1 )
           .agg(myAgg.as[String])

// ERROR: object creation impossible, since: it has 3 unimplemented members. 
// ERROR: /** 
// ERROR: *  As seen from <$anon: org.apache.spark.sql.expressions.Aggregator[(Int, String),String,String]>, 
// ERROR: *  the missing signatures are as follows. For convenience, these are usable as stub implementations. 
// ERROR: */ 
// ERROR: def bufferEncoder: org.apache.spark.sql.Encoder[String] = ??? 
// ERROR: def outputEncoder: org.apache.spark.sql.Encoder[String] = ???
```
So as seen, we get a compile-time error. **We are missing 2 methods implementations!** So whats an Encoder?

#### Encoder

Encoders are what **convert your data between JVM Objects and Spark SQL's specialized internal tabular representation**. (The encoded data is then serialized by the Tungsten off-heap Serializer)

**They are required by all `Datasets`!

Encoders are highly specialized, optimized code generators that generate custom bytecode for serialization and de-serialization of data.

The serialized data is stored using Spark internal Tungsten binary format, allowing for operations on serialized data and improved memory utilization.

**What sets them apart from regular Java or Kryo serialization**

* Limited to and optimal for primitives and case classes, Spark SQL datatypes, which are well understood.
* The contain schema information, which makes these highly optimized code generators possible, and enables optimization based on the shape of the data. Since Spark understands the structure of data in Datasets, it can create a more optimal layout in memory when caching Datasets.
* Uses significantly less memory than Kryo/Java serialization
* 10x faster than Kryo serialization, Java serialization orders of magnitude slower.

**To ways to introduce encoders:**

* Automatically (generally the case) via implicits from a `SparkSession`. i.e. `import spark.implicits._`
* Explicitly via `org.apache.spark.sql.Encoder` which contains a large selection of methods for creating `Encoder`s from  Scala primitive types and `Product`s.

**Some examples of 'Encoder' creation methods in 'Encoders':**

* `INT/LONG/STRING etc` for `nullable` primitives
* `scalaInt/scalaLong/scalaByte etc` for Scala primitives
* `product/type` for Scala's `Product` and tuple types.

**Example**: Explicitly creating Encoders:

```scala
Encoders.scalaInt // Encoder[Int]
Encoders.STRING // Encoder[String]
Encoders.product[Person] // Encoder[Person], where Person extends Product or is a case class
```

Now, lets get back to the example if emulating `reduceByKey` with an Aggregator where we had an error regarding Encoders:

```scala
val keyValues = List( (3, "Me"),(1, "Thi"),(2, "Se"),(3, "ssa"),(1, "sIsA"),(3, "ge:"),(3, "-)"),(2, "cre"),(2, "t") )
val keyValuesDS = keyValues.toDS

val myAgg = new Aggregator[(Int, String), String, String] {
    def zero: String  = ""                                       // Initial value - empty string
    def reduce(b: String, a: (Int, String)): String  = b + a._2  // Add an element to the running total
    def merge(b1: String, b2: String): String  = b1 + b2         // Merge Intermediate values
    def finish(r: String): String  = r                           // Return final result
    override def BufferEncoder: Encoder[BUF] = ???          
    override def outputEncoder: Encoder[OUT] = ???
}.toColumn

keyValuesDS.groupByKey( pair => pair._1 )
           .agg(myAgg.as[String])
```
After defining the encoders:

```scala
val keyValues = List( (3, "Me"),(1, "Thi"),(2, "Se"),(3, "ssa"),(1, "sIsA"),(3, "ge:"),(3, "-)"),(2, "cre"),(2, "t") )
val keyValuesDS = keyValues.toDS

import org.apache.spark.sql.Encoder
import org.apache.spark.sql.Encoders

val myAgg = new Aggregator[(Int, String), String, String] {
   def zero: String = ""                                        // Initial value - empty string
    def reduce(b: String, a: (Int, String)): String = b + a._2  // Add an element to the running total
    def merge(b1: String, b2: String): String = b1 + b2         // Merge Intermediate values
    def finish(r: String): String = r                           // final value
    override def bufferEncoder: Encoder[String] = Encoders.STRING
    override def outputEncoder: Encoder[String] = Encoders.STRING
}.toColumn

keyValuesDS.groupByKey( pair => pair._1 )
           .agg(myAgg.as[String]).show

// +-----+--------------------+
// |value|anon$1(scala.Tuple2)|
// +-----+--------------------+
// |    1|             ThisIsA|
// |    3|          Message:-)|
// |    2|              Secret|
// +-----+--------------------+

```

## Common Dataset Actions

```scala
collect(): Array[T]
// Returns an array that contains all the Rows in the Dataset

count(): Long
// Returns the no. of rows in the Dataset

first(): T 
head(): T
// Returns the first row in the Dataset

foreach(f: T => Unit): Unit
// Applies a function f to all the rows in the Dataset

reduce(f: (T, T) => T): T
// Reduces the elements of this Dataset using the specified binary function

show(): Unit
// Displays the top 20 rows of the Dataset in a tabular form

take(n: Int): Array[T]
// Returns the first n rows in the Dataset
```

## When to use Datasets vs DataFrames vs RDDs?

Use **Datasets** when:

* you have structured/semi-structured data
* you want typesafety
* you need to work with functional APIs
* You need good performance, but it doesn't have to be the best

Use **DataFrames** when:

* you have structured/semi-structured data
* You want the best performance, automatically optimized for you

Use **RDDs** when:

* you have unstructured data
* you need to fine-tune and manage low-level details of RDD computations
* you have complex dat types that cannot be serialized with `Encoder`s

Performance wise, RDD < Dataset < DataFrame.

## Dataset Limitations

### 1. Catalyst Can't Optimize All Operations

Take filtering as example:

* **Relational filter operation:** E.g. `ds.filter($"city".as[String] === "Boston")`. Performs best because you are explicitly telling Spark which columns/attributes and conditions are required in your filter operation. With information about the structure of the data and the structure of computations, Spark's optimized knows it can access only the fields involved in the filter without having to instantiate the entire data type. Avoids dat moving over the network. **Catalyst optimizes this case**.

* **Functional filter operation:** E.g. `ds.filter(arg => arg.city == "Boston")`. This is the same filter written with a function literal. It is opaque to Spark - it is impossible for Spark to introspect the lambda function. All Spark knows is that you need a (whole) record marshaled as a Scala object in order to return true or false, requiring Spark to do potentially a lot more work to meet that implicit requirement. **Catalyst cannot optimize this case**.

**Takeaway:**

* When using `Dataset`s with higher-order functions like `map`, you miss out on many Catalyst optimizations.
* When using `Dataset`s with relational operations like `select`, you get all of Catalyst optimizations.
* Though not all operations on `Datasets` benefit from Catalyst's optimizations, Tungsten is still always running under the hod of `Dataset`s, storing and organizing data in a highly optimized way, which can result in large speedups over RDDs.

### 2. Limited Data Types:

If your data cannot be expressed by `case class`es/`Product`s and standard Spark SQL data types, it may be difficult to ensure that a Tungsten encoder exists for your data type.

### 3. Requires Semi-structured/Structured Data

If your unstructured data cannot be reformulated to adhere to some kind of schema, it would be better to use RDDs.