What happens when we do a `groupBy` or a `groupByKey` on a RDD? (Remember that our data is distributed on multiple nodes).

```scala
val list = List( (1, "one"),(2, "two"),(3, "three") )
val pairs = sc.parallelize(list)
pairs.groupByKey()
// > res2: org.apache.spark.rdd.RDD[(Int, Iterable[String])] 
//         = ShuffledRDD[16] at groupByKey at <console>:37
```

We typically have to move data from one node to another to be "grouped" with its "key". Doing this is called **Shuffling**. We never call this shuffle method directly, but it happens behind to curtains for some other functions as above. 

This might be very expensive for performance because of **Latency**!

**`groubByKey` results in one key-value pair per key. This single key-value pair cannot span across multiple worker nodes.**

### Example

```scala
// CFF is a Swiss train company
case class CFFPurchase(customerId: Int, destination: String, price: Double)
// Assume that we have an RDD of purchases users of CFFs mobile app have made in the past month
val purchasesRdd: RDD[CFFPurchase] = sc.textFile(...)
```

**Goal**: Calculate how many trips, and how much money was spent by each individual customer over the course of the month.

```scala
val purchasesPerMonth = purchasesRdd.map( p => (p.customerId, p.price) ) // pair RDD
                                    .groupByKey()                        // RDD[K, Iterable[V]] i.e RDD[p.customerId, Iterable[p.price]]
                                    .map( p => (p._1, (p._2.size, p._2.sum)) )
                                    .collect()
```

Example dataset for the above problem:

```scala
val purchases = List( CFFPurchase(100, "Geneva", 22.25),
                      CFFPurchase(100, "Zurich", 42.10),
                      CFFPurchase(100, "Fribourg", 12.40),
                      CFFPurchase(100, "St.Gallen", 8.20),
                      CFFPurchase(100, "Lucerne", 31.60),
                      CFFPurchase(100, "Basel", 16.20) )
```
How would this look on a cluster?

Lets say we have 3 nodes and our data is evenly distributed on it, so above operations look like this:

![shuffline](https://github.com/rohitvg/scala-spark-4/blob/master/resources/images/shuffline.png)

This shuffling is very expensive because of **Latency**.

### Can we do a better job?

Perhaps we can reduce before we shuffle. This could greatly reduce the amount of data we send over network. 

To do this we use `reduceByKey` that we have [seen earlier](https://github.com/rohitvg/scala-spark-4/wiki/Pair-RDDs:-Transformations-and-Actions#reducebykey-transformation).

```scala
val purchasesPerMonth = purchasesRdd.map( p => (p.customerId, (1, p.price)) ) // pair RDD
                                    .reduceByKey( (v1, v2) => (v1._1 + v2._1, v1._2 + v2._2) )
                                    .collect()                        
```
What will this look like on the cluster?

![shuffling_2](https://github.com/rohitvg/scala-spark-4/blob/master/resources/images/shuffline_2.png)

**Note**: Here we shuffle considerable less amount of data, just by using `reduceByKey` instead of doing a `groupByKey` followed by `map`.

**Benefits of this approach:**

* By reducing the dataset first, the amount of data sent over the network during the shuffle is greatly reduced. Thus performance gains are achieved!

**This is because when using `groupByKey`, it requires collecting all key-valu pairs with the same key on the same machine.**

**Question:** How does Spark know which key to put on which machine?

By default, Spark uses **hash partitioning** to determine which key-value pair should be sent to which machine.



