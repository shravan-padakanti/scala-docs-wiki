In this session we're going to talk about evaluation in Spark and in particular, reasons why Spark is very unlike Scala Collections.

## Why is Spark Good for Data Science?

We saw Spark has **Transformation _laziness_** and **Action _eagerness_** which allow it to aggressively minimize network operations for addressing latency concerns. Also this allows Spark to use more **in-memory computation** which is much faster.

Most data science problems involve iteration. 

First lets look at iteration in Hadoop:

![iteration_in_hadoop]()

As seen each iteration has a `map/reduce` step, and result data is written between each iteration, also read by the next iteration. Thus there is a lot of time spent in IO. Spark can avoid upto 90% of this time. 

Iteration in Spark:

![iteration_in_spark]()

As seen, data is read once, and then it stays on memory where its iterated over and over. This also provides good fault tolerance due to significantly reduced IO.

# Iteration example: Logistic Regression

Logistic Regression is an algorithm used for classification (classify given data-points into clusters) and here the classifiers weights are iteratively updated based on a training dataset. 

Here is the algorithm:

```scala
val points = sc.textFile(...).map(parsePoint)
var w = Vector.zeros(d)
for (i <- 1 to numIterations) {
    val gradient = points.map { p =>
        (1 / (1 + exp(-p.y * w.dot(p.x))) -1_ * p.y * p.y
    }.reduce(_ + _)
    w -= alpha * gradient
}
```
As seen above, **action** `reduce` is being called inside the for-loop. Hence the **transformation** `map` will be applied twice times the number of iterations (twice because there is one `map` outside the for-loop which is not applied as we have already learned.). The use of inner `map` transformation is intentional, but the outer `map` transformation gets unintentionally used mulitple times inside the iteration. So here we are doing IO like Hadoop, lots of it during every iteration.

**Note**: Spark allows you to control  what is cache in memory by using `persist()` or `cache()` on a `RDD`.

### Caching and Persistance

Lets [revisit our previous example](https://github.com/rohitvg/scala-spark-4/wiki/RDDs:-Transformation-and-Action#benefits-of-laziness-for-large-scale-data):

```scala
val lastYearsLogs: RDD[String] = ...
val logsWithErrors = lastYearsLogs.filter(_.contains("ERROR"))
val firstLogsWithErrors = logsWithErrors.take(10)  // action 1
val numErrors = logsWithErrors.count()             // action 2
```
Here an action is applied on `logsWithErrors` twice. Hence the transformation is alos applied twice. To resolve this, we modify the above as:

```scala
val lastYearsLogs: RDD[String] = ...
val logsWithErrors = lastYearsLogs.filter(_.contains("ERROR")).persist() // <------------------ persist
val firstLogsWithErrors = logsWithErrors.take(10)  // action 1
val numErrors = logsWithErrors.count()             // action 2
```

Here we cache `logsWithErrors` in memory. So its computed only once, and then for rest of the references, it is used from the memory.

Similarly, we can also re-write the Logistic Regression algorithm, so that the outer `map` transformation is only computed the first time and persisted.

```scala
val points = sc.textFile(...).map(parsePoint).persist() // <------------------ persist 
var w = Vector.zeros(d)
for (i <- 1 to numIterations) {
    val gradient = points.map { p =>
        (1 / (1 + exp(-p.y * w.dot(p.x))) -1_ * p.y * p.y
    }.reduce(_ + _)
    w -= alpha * gradient
}
```

**There are many ways to configure how the data is persisted**:

* in memory as regular java objects (just like a regular Java program - least used elements are evaculated by JVM)
* on disk as regular java objects
* in memory as serialize Java objects (more compact since uses byte arrays)
* on disk as serialize Java objects (more compact since uses byte arrays)
* both in memory and on disk (spill over to disk to avoid re-computation)

`cache()`: Shorthand for using default storage lever, which is in memory only as regular Java objects.

`persist(...)`: Persistence can be customized with this methods. Pass the storage level you'd like as a parameter. Storage levels:

```
Level                   Space used          CPU time          In memory          On disk

MEMORY_ONLY (default)   High                Low               Y                  N
MEMORY_ONLY_SER         Low                 High              Y                  N
MEMORY_AND_DISK*@       High                Medium            Some               Some
MEMORY_AND_DISK_SER*&   Low                 High              Some               Some
DISK_ONLY               Low                 High              Y                  N
```

*@ = Spills to disk if too much data to fit in memory
*& = Spills to disk if too much data to fit in memory. Stores serialized representation in memory.

##  Key Takeaway

Despite similar-looking API to Scala Collections, the evaluation of Spark RDDs is very unlike them.

**People (newcomers) unknowing re-evaluate several transformations when caching could be used.**
