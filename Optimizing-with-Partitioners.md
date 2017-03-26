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