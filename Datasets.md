When we call a `collect()` on DataFrames,  we get an `Array[org.apaceh.spark.sql.Row]`.

Going back to the [listings example](https://github.com/rohitvg/scala-spark-4/wiki/DataFrames-(1)#example):

```scala
case class listings(street: String, zip: Int, price: Int)
val listingDF = ...

import org.apache.spark.sql.functions._

val averagePricesDF = listings.groupBy($"zip")
                              .avg($"price")

val averagePrices = averagePricesDF.collect()
// averagePrices : Array[org.apaceh.spark.sql.Row]
```

What is this?

Since DataFrames are made up of `Row`s , and don't have any type information, we have to cast things:

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

Based on this we update our cast:

```scala
val averagePricesAgain = averagePrices.map {
                    row => ( row(0).asInstanceOf[Int], row(1).asInstanceOf[Double] )
}
// averagePricesAgain : Array[(Int, Double)]
```

**Wouldn't it be nice if we could have both: Spark SQL Optimizations and typesafety?**

This is where **Datasets** come into play.

## Datasets

**DataFrames are actually DataSets**

```scala
type DataFrame = Dataset[Row]
```

What the heck is a Dataset?

* **Typed** distributed collections of data.
* unify the `DataFrame` and the `RDD` APIs. **Mix and Match!**
* require structured/semi-structured data. Schemas and Encoders are core part of Datasets.

**Things of Datasets as a compromise between RDDs and DataFrames.** You get more type information on Datasets than on DatFrames, and you get more optimizations than you get on RDDs.

**Example**

Going back to the [listings example](https://github.com/rohitvg/scala-spark-4/wiki/DataFrames-(1)#example):

Lets calculate the average home price per zipcode with Datasets this time. Assuming listingDS is of type Dataset[Listing]:

```scala
listingsDS.groupByKey(1 => 1.zip)         // looks like groupByKey on RDDs!
          .agg(avg($"price").as[Double])  // looks like our DataFrame operators!
```

We can freely mix APIs!

So, **Datasets are something in the middle between DataFrames and RDDs**. 

* We can still use relational `DataFrame` operations that we learned on `DataSet`s
* `DataSet`s add more typed operations that can be used as well
* `DataSet`s let you use higher-order functions like map, flatMap, filter again!

So Datasets can be used when we want a mix of functional and relational transformation while benefitting from some of the optimizations on DataFrames.

And we've **almost** got a type safe API as well.

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


