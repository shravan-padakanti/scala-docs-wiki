Here, we're going to focus on distributed key values which are a popular way of representing and organizing large quantities of data in practice. In Spark, we called these distributed key-value pairs **Pair RDDs**.

In single-node Scala, key-value pairs can be thought of as **maps**, or in someother programming languages as **dicitonaries**. **Map-Reduce** is also based on key-value pairs of large distributed datasets.

They are useful because they allow us to act on each key in parallel or regroup data across the network.

An RDD parameterized by a pair are treated as Pair RDD by spark:

```scala
RDD[(k, v)] // <------- treated specially by Spark
```

Pair RDDs can be created from already existing regular RDDs for example by using the `map` operation on the regular RDD:

```scala
val rdd: RDD[WikipediaPage] = ...
val pairRdd = rdd.map( wikipediapage => (wikipediapage.title, wikipediapage.text) ) 
```

Pair RDDs have additional, specialized methods for working with data associated with keys. Some of the commonly used are:

```scala
def groupByKey(): RDD[(K, Iterable[V])
def reduceByKey(func: (V, V) => V): RDD[(K, V)]
def join[W](other: RDD[(K, W)]): RDD[(K, (V, W))]
```


