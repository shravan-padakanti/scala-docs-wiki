Here we will look at different kind of operations and actions that we can perform on RDDs. 

Recall transformers and accessors from **Scala** sequential and parallel collections.

**Transformers**: Return new collections as results. (Not single values, thus transform one collection into another)
Examples: `map`, `filter`, `flatMap`, `groupBy`

```scala
map(f: A => B): Traversable[B]
```
**Accessors**: Return single values as results.
Examples: `reduce`, `fold`, `aggregate`.
```scala
reduce(op: (A, A) => A): A
```

In **Spark**, we have these counterparts:

**Transformers**: Return new ~~collections~~ RDDs as results.<br/>**They are _lazy_, their result RDD is not immediately computed**.

**Actions**: Compute a result based on an RDD, and its either returned or saved to an external storage system like HDFS.<br/>**They are _eager_, their result is immediately computed**. So if RDD is not returned as a result, the given function an action.

**Laziness/eagerness is how we can limit network communication using the programming model**. 

These properties is how *Spark* provides the benefits [mentioned earlier](https://github.com/rohitvg/scala-spark-4/wiki/Latency#question-how-do-these-numbers-affect-big-data-processing) and is able to aggresively reduce the required network communication, thus addressing latency. This example makes it clear:

Consider a **transformation**:
```scala
val largeList: List[String] = ...
val wordsRdd: RDD[String] = sc.parallelize(largeList)   // sc is the SparkContext
val lengthsRdd: RDD[Int] = wordsRdd.map(_.length)
```
What has happened on the cluster at this point?
**Nothing**: Execution of map (a transformation) is deferred as the **transformations** which are **lazy**, Spark just keeps track of the transformation.

Now we add an **action** to the above:
```scala
val totalChars = lengthsRdd.reduce( _ + _ )
```
Now the transformation is applied on the dataset and the the **action** which is **eager** is applied on the result of that.

Thus, as we [saw previously](https://github.com/rohitvg/scala-spark-4/wiki/Latency#question-how-do-these-numbers-affect-big-data-processing), Spark minimizes latency by aggresively minimizing the network communications by using **lazy transformations** and **eager actions**.

So people erroneously think that after applying a transformation, the result has been computed, whereas in reality, the result is only computed when an *action* is used.

### Common Transformations in the Wild

* `map[B](f: A => B): RDD[B]`: Apply function `f` to each element in RDD and return an RDD of the result
* `flatMap[B](f: A => TraversableOnce[B]): RDD[B]`: Apply function `f` to each element in RDD and return an RDD of the contents of the iterators returned
* `filter(pred: A => Boolean): RDD[A]`: apply `pred` function to each element in RDD and return an RDD of elements that pass the predicate condition.
* `distinct(): RDD[B]`: return RDD with duplicates removed.

### Common Actions in the Wild

* `collect(): Array[T]`: returns all elements in RDD
* `count(): Long`: returns num of elements in RDD
* `take(num: Int): Array[T]`: returns first `num` elements of RDD
* `reduce(op: (A,A) => A): A`: combine the elements in the RDD together using the given `op` functions and return result.
* `foreach(f: T => Unit): Array[T]`: apply function to each element in the RDD.

# Another example

Given an `RDD[String]` which contain gigabytes of logs. Each element of this RDD represents one line of logging. Assuming the dates come in the form `YYYY-MM-DD:HH:MM:SS`, and errors are logged with prefix "ERROR", how would you determine the number of errors that were logged in December 2016?

```scala
val lastYearsLogs: RDD[String] = ...
val numErrorsLoggedInDec = lastYearsLogs.filter(arg => arg.contains("2016-12") && arg.contains("ERROR")) // line 1
                                        .count // line 2
// line 1: A computation that we know we're going to eventually do, but we haven't started it yet - lazy.
// line 2: Actually gives the order to Spark to send this function over the network to all of the little individual machines to do their computations, and then to add them up and send back the results, the count call. And to aggregate it, combine it all up, so that you have one integer or one long with the number of errors in the logs
```

### Benefits of Laziness for Large-Scale Data

Spark computes RDDs the first time they are used in an action. This helps when processing large amounts of data. Consider the example from above:

```scala
val lastYearsLogs: RDD[String] = ...
val firstLogsWithErrors = lastYearsLogs.filter(_.contains("ERROR")).take(10)
```

The execution of `filter` transformation is deferred until the `take` action is applied.

Spark leverages this by analyzing and optimizing the **chain of operaitons** before executing it.

Spark will not compute intermediate RDDs. Instead, as soon as 10 elements of the filtered RDD have been computed, we are done. At this point, Spark stops working, saving time and space by not computing unused result of `filter`.

### Transformation of 2 RDDs

* `union(other: RDD[T]): RDD[T]`: Returns an RDD containing elements from both RDDs
* `intersection(other: RDD[T]): RDD[T]`: Returns an RDD containing elements common to both RDDs
* `subtract(other: RDD[T]): RDD[T]`: Returns the RDD with contents of other RDD removed.
* `cartesian(other: RDD[T]): RDD[T]`: Returns an RDD that is cartesian product with the other RDD.

### Other useful RDD Actions

RDDs also contain other actions unrelated to Scala collections, but which are useful when dealing with distributed data

* `takeSample(withRep1: Boolean, num: Int): Array[T]`: returns an array with random sample of `num` elements of the dataset, with or without replacement.
* `takeOrdered(num: Int)(implicit ord: Ordering[T]): Array[T]`: returns the first n elements of the RDD using either rheir natural order or a custom comparator.
* `saveAsTextFile(path: String): Unit`: write elements of the dataset as a text file in the local filesystem or HDFS.
* `saveAsSequenceFile(path: String): Unit`: write elements of the dataset as a Hadoop sequence file in the local filesystem or HDFS.
