Last lecture, we learned about the **hash partitioners** and **range partitioners**, and also learned about what kind of operations may introduce new partitioners or which may discard custom partitioners.

### Why would we want to repartition the data?

It can bring substantial performance gains, especially in face of shuffles.

[We saw](https://github.com/rohitvg/scala-spark-4/wiki/Shuffling:-What-it-is-and-why-it's-important#example) how using `reduceByKey` instead of `groupByKey` localizes data better due to different partitioning strategy and thus reduces latency to deliver performance gains.

By manually repartition the data, for the same example, we can improve the performance even further. By usnig range partitioners we can optimize the use of `reduceByKey` in that example so that it does not involve any shuffling over the network at all!

```scala
/* Same as before */

// CFF is a Swiss train company
case class CFFPurchase(customerId: Int, destination: String, price: Double)
// Assume that we have an RDD of purchases users of CFFs mobile app have made in the past month
val purchasesRdd: RDD[CFFPurchase] = sc.textFile(...)
val pairs = purchasesRdd.map( p => (p.customerId, p.price) )

/* NEW! */

val tunedPartitioner = new RangePartitioners(8, pairs) // 8 partitions
val partitioned = pairs.partitionBy(tunedPartitioner).persist()

/* Same as before */

val purchasesPerMonth = partitioned.map( p => (p._1, (1, p._2) ) //uses partitioned instead of purchasesRDD
                                   .reduceByKey( (v1, v2) => (v1._1 + v2._1, v1._2 + v2._2) )
                                   .collect()
```

This gives a 9 times speedup in practical tests.

### Partitioning Data: `partitionBy` (Pg.61-64: Learning Spark book)

Consider a media application that keeps a large table of user information in memory. Here users can subscribe to different topics.

* `userData`: **BIG** Pair RDD, contains `(UserID, UserInfo)` pairs. `UserInfo` contains a list of topics the user is subscribed to.

The application periodically combines this table with a smaller file representing events that happened in the past five minutes:

* `events`: **SMALL** Pair RDD, containing `(UserID, LinkInfo)` pairs for users who have clicked a link on a website in those five minutes.

For example, we may wish to count how many users visited a link that was not to one of their subscribed topics. We can perform this combination with Spark's `join` operation, which can be used to group the 1UserInfo` and `LinkInfo` w.r.t. the `UserID` key.

Here is a program which does that:

```scala
val sc = new SparkContext(...)
val userData = sc.sequenceFile[UserIF, UserInfo]("hdfs://...").persist()

def processNewLogs(logFileName: String) {
    val events = sc.sequenceFile[UserID, LinkInfo](lofFileName)
    val joined = userData.join(events) // RDD of (UserId, (UserInfo, LinkInfo))
    // find the no.of links that the user clicks on that he is not subscribed to
    val offTopicVisits = joined.filter {
        case (userId, (userInfo, linkInfo)) => !userInfo.topics.contains(linkInfo.topic)
    }.count()
    println("No. of visits to non-subscribed topics: " + offTopicVisits)
}
```
This code works as desired, but **it is very inefficient**: The `join` operation is called everytime `processNewLogs` is invoked, and it doe not know anything about how the keys are partitioned in the datasets. By default, this operation will hash all the keys of both datasets (`userData` and `events`) sending elements with the same key hash across the network to the same machine, and then join the elements with the same key on that machine, although that data already existed since the pairRdd `userData` does not change.

Fixing this is easy! Use `partitionBy` on the **Big** `userData` PairRDD at the start of the program:

```scala
// change this
val userData = sc.sequenceFile[UserIF, UserInfo]("hdfs://...").persist()

// to this
val userData = sc.sequenceFile[UserIF, UserInfo]("hdfs://...")
                 .partitionBy(new HashPartitioner(100)) // create 100 partitions
                 .persist()
```
Spark will now know that `userData` is **hash-partitioned**, and when `join` is called on it, Spark will take advantage of this information. So Spark will only **shuffle** the `events` Pair Rdd, sending events with each particular `UserId` to the machine that contains the corresponding hash partition of `userData`.

### How do I know when a shuffle will occur?

**Rule of Thumb**: a shuffle _can_ occur when the resulting RDD depends on other elements from the same RDD or another RDD.

We can also figure out if a shuffle has been planned or executed via:

1. The return type of certain transformations. E.g. 
    ```scala
    org.apache.spark.rdd.RDD[(String, Int)] = ShuffleRDD[366] //shuffle has happended or planned
    ```

2. Using function `toDebugString` to see its execution plan: 
    ```scala
    partitioned.reduceByKey((v1, v2) => (v1._1 + v2._1, v1._2 + v2._2)).toDebugString
    res9: String = 
    (8) MapPartitionRDD[633] at reduceNyKey at <console>:49 []
    | ShuffledRDD[615] at partitionBy at <console>:48 []
    |    CachedPartitions: 8; MemorySize: 1754.8 MB, DiskSIze: 0.0 B.
    ```

Operations that **might** cause a shuffle: 

* `cogroup`
* `groupWith`
* `join`
* `leftOuterJoin`
* `rightOuterJoin`
* `groupByKey`
* `reduceByKey`
* `combineByKey`
* `distinct`
* `intersection`
* `repartition`
* `coalesce`

### Common scenarios where Network Shuffle can be avoided using Partitioning

1. `reduceByKey` running on a pre-partitioned RDD will cause the values to be computed **locally**, requiring only the final reduced value to be sent from worker to the driver.
2. `join` called on 2 RDDs that are pre-partitioned with the same partitioner and cached on the same machine will cause the join to be computed **locally**, with no shuffling across the network.

### Key Takeaways

**How the data is organized on the cluster, and what operations we are doing on it matters!**