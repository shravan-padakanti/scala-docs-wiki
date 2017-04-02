When we call a `collect()` on DataFrames,  we get an `Array[org.apaceh.spark.sql.Row]`.

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
