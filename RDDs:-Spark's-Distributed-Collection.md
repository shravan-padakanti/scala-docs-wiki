As we [saw previously](https://github.com/rohitvg/scala-spark-4/wiki/Data-Parallel-to-Distributed-Data-Parallel#apache-spark), **RDDs** are **Spark's Parallel Distributed Collections** abstractions and they're really at the core of spark.

RDDs seem a lot like **immutable** sequential or parallel Scala collections, as it offers similar functions like `map, flatMap, filter, reduce, fold, aggregate`, etc. 

```scala
abstract class RDD[T] {
    def map[U](f: T => U): RDD[U] = ...
    def flatMap[U](f: T => TraversableOnce[U]): RDD[U] = ...
    def filter[U](f: T => Boolean): RDD[U] = ...
    def reduce[U](f: (T, T) => T): T = ...
    ...
}
```
Most operations on RDDs , like Scala's immutable List, and Scala's parallel collections, are higher order functions (i.e. methods that work on RDDs, taking a function as an argument and which typically return RDDs).

But as seen above, the signatures are also very similar, differing in the respective return types.
```scala
map[B](f: A => B): List[B] // Scala List
map[B](f: A => B): RDD[B]  // Spark RDD

flatMap[B](f: A => TraversableOnce[B]): List[B] // Scala List
flatMap[B](f: A => TraversableOnce[B]): RDD[B]  // Spark RDD

filter(pred: A => Boolean): List[A] // Scala List
filter(pred: A => Boolean): RDD[A]  // Spark RDD

reduce(op: (A, A) => A): A // Scala List
reduce(op: (A, A) => A): A // Spark RDD

fold(z: A)(op: (A, A) => A): A // Scala List
fold(z: A)(op: (A, A) => A): A // Spark RDD

aggregate[B](z: => B)(seqop: (B, A) => B, combop: (B, B) => B): B // Scala
aggregate[B](z: B)(seqop: (B, A) => B, combop: (B, B) => B): B // Spark RDD
```
Only difference is `aggregate`, where in case of Scala it has `0` as its "by-name" parameter, where as in Spark that is not the case. This is since, in case of Spark we are dealing with distributed data and we don't want to evaluate it more than once as it may be over the network.

**Bottomline**: Using RDDs in Spark feels a lot like normal Scala sequential/parallel collections, with the added knowledge that your data is distributed across several machines. Eg. Given, `val encyclopedia: RDD[String]`, say we want to search all of encyclopedia for mentions of "EPFL", and count the number of pages that mention EPFL:
```scala
val result = encyclopedia.filter(page => page.contains("EPFL")).count()
```

### Example: Word Count

```scala

// Create an RDD
val rdd : RDD[String] = spark.textFile("hdfs://..") 

val count : RDD[String] = rdd.flatMap(arg => arg.split(" "))   // separate text into words
                             .map(word => (word, 1))           // pair each word with 1
                             .reduceByKey(_ + _)               // sum up the 1s in the pairs
```

### How to create an RDD

2 ways: 

1. **Transforming an existing RDD**: Just like a call to `map` on a `List` returns a new `List`, many higher-order functions defined on a `RDD` return a new `RDD`
2. **From a `SparkContext` (or `SparkSession`) object**: The `SparkContext` (or `SparkSession`) is our **handle to the Spark cluster**. It represents the connection between the Spark cluster and your running application. It defines a handful of methods which can be used to create and populate a `RDD`:
    1. `parallelize`: convert a local Scala collection to an RDD
    2. `textFile`: read a text file from HDFS or local filesystem and return a `RDD[String]`.