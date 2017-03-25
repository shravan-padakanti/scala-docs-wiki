[Just like in regular RDDs](https://github.com/rohitvg/scala-spark-4/wiki/RDDs:-Transformation-and-Action#common-transformations-in-the-wild), the operations on Pair RDDs can be broken down into: 

**Transformations**:

* `groupByKey`
* `reduceByKey`
* `mapValues`
* `keys`
* `join`
* `leftOuterJoin`/`rightOuterJoin`

**Actions**:

* `countByKey`

### `groupByKey`  (transformation)

In Scala, we have a `groupBy` operation. It breaks up a collections into 2 or more collections based on the function passed to it. The result of the argument function is the key, and the elements that yield the result when passed to the function form the elements of the collection which is the value to that key.

```scala
def groupBy[K](f: A => K): Map[K, traversable[A]]

/* Eg. 1 */
val numbers = List(1,5,1,6,5,2,1,9,2,1)
val group = numbers.groupBy(x => x) 
// Map[Int,List[Int]] = Map(5 -> List(5, 5), 1 -> List(1, 1, 1, 1), 6 -> List(6), 9 -> List(9), 2 -> List(2, 2))
val group2 = numbers.groupBy(x => x+1)
// Map(10 -> List(9), 6 -> List(5, 5), 2 -> List(1, 1, 1, 1), 7 -> List(6), 3 -> List(2, 2))

/* Eg. 2 */
val ages = List(2, 52, 44, 23, 16, 11, 82)
val groups = ages.groupBy{ arg => ( if(arg >= 18) "adult" else "child" ) }                          
// Map(adult -> List(52, 44, 23, 82), child -> List(2, 16, 11))
```

In Spark, `groupByKey` works on Pair RDDs, and since we Pair RDDs are already in a key value form, it does not need a function as an argument. So we just want to 

```scala
def groupByKey(): RDD[(K, Iterable[V])]

/* Eg */
case class Event(Organizer: String, name: String, budget: Int)
val rdd = sc.parallelize(...)
val eventsRdd = rdd.map(event => (event.organizer, event.budget)) // Pair RDD
val groupedRDD = eventsRdd.groupByKey() // nothing happens! since its lazy.
// Now to evaluate we call an action on it, and we find it grouped with organizer as the key, and different budgets from that organizer as values.
groupedRDD.collect().foreach(println)
// (Organizer1, CompactBuffer(42000))
// (Organizer2, CompactBuffer(20000, 44400, 87000))
```

### `reduceByKey` (transformation)

It is a combination of `groupByKey` followed by `reduce` on values of each grouped collection. It is more efficient than using the both separately.

Note that it takes in the value and returns a value. So if the value is a pair, it also returns a pair.

``scala
def reduceByKey( func(V, V) => V ): RDD[(K, V)] // V corresponds to the values of Pair RDD, we only operate on the value since a pair RDD is in the form of Key Values.

/* Eg. Going back to the last events example, if we want to calculate the total budget per organization, then: */
val eventsRdd = rdd.map(event => (event.organizer, event.budget)) // Pair RDD
val totalBudgetsRdd = eventsRdd.reduceByKey( _ + _ ) // at this point we already have keys and values. So reduceByKey means reduce the values corresponding to the given key using the given function. Its a transformation, lazy, so nothing happens!
totalBudgetsRdd.collect().foreach(println)
// (Organizer1, 42000)
// (Organizer2, 151400)
```

### `mapValues` (transformation)

It applies the given function to only the values in a Pair RDD i.e. transforms `RDD[(K, V)]` to `RDD[(K, U)]`.

```scala
def mapValues[U](f: V => U): RDD[(K, U)]
```

### `keys` (transformation)

Returns an RDD by collecting all the keys from the tuple of the Pair RDD

```scala
def keys: RDD[K]
```

### `countByKey` (action)

It counts the no. of elements per key in a Pair RDD and returns a regular Scala `Map` of key against the count. Its an **action** so its eager.

```scala
def countByKey(): Map[K, Long]
```

# Full Combined Example

Given the events RDD, we use the above methods to compute average budget per even organizer

Input file is events.dat:
```
Organizer1,A,20000
Organizer1,B,10000
Organizer2,C,30000
Organizer1,D,20000
Organizer3,E,10000
Organizer1,F,20000
Organizer1,G,20000
Organizer2,H,10000
Organizer3,I,20000
Organizer1,J,20000
```

```scala
package com.example.spark

import org.apache.spark.rdd.RDD

case class Event(organizer: String, name: String, budget: Int)

object Main extends App {

  val sc = SparkHelper.sc
  val rdd = sc.textFile("src/main/resources/events.dat").map(
    stringArg => {
      val array = stringArg.split(",")
      new Event(array(0), array(1), array(2).toInt)
    })
    
  /* Create a Pair RDD */
  println("Eg. 1) Create a Pair RDD")
  // create a pairRdd from rdd by pairing the organizer with the event budget
  val pairRdd = rdd.map(event => (event.organizer, event.budget)) // Pair RDD
  pairRdd.collect().foreach(println)
  
  /* Group the Pair RDD using orgainzer as the key */
  println("Eg. 2) Group the Pair RDD using orgainzer as the key")
  val groupedRdd = pairRdd.groupByKey()
  groupedRdd.collect().foreach(println)
  
  /* Instead of grouping, reduce the pair rdd to organizer with the total budget */
  println("Eg. 3) Instead of grouping, reduce the pair rdd to organizer with the total budget")
  val reducedRdd = pairRdd.reduceByKey(_ + _)
  reducedRdd.collect().foreach(println)
  
  /* Instead of just reducing to organizer with the total of the budget, reduce 
   * to organizer and avg. budget */
  println("Eg. 4) Instead of just reducing to organizer with the total of the budget, reduce to organizer and avg. budget")
  val coupledValuesRdd = pairRdd.mapValues( v => (v, 1) )
  println("\ncoupledValuesRdd: ")
  coupledValuesRdd.collect().foreach(println)
  
  // since the value is a pair at this point, it will also return a pair as a value 
  // This results in (organizer, (total budget, total no.of events) )
  val intermediate = coupledValuesRdd.reduceByKey( (v1, v2) => ( (v1._1 + v2._1), (v1._2 + v2._2) ) )
  println("\n intermediate : ")
  intermediate.collect().foreach(println)
  
  val averagesRdd = intermediate.mapValues( pair => pair._1/pair._2 )
  println("\n averagesRdd : ")
  averagesRdd.collect().foreach(println)

}
```

Output: 

```
Using Spark's default log4j profile: org/apache/spark/log4j-defaults.properties
17/03/25 15:35:07 INFO SparkContext: Running Spark version 1.4.0
17/03/25 15:35:07 WARN NativeCodeLoader: Unable to load native-hadoop library for your platform... using builtin-java classes where applicable
17/03/25 15:35:12 INFO SecurityManager: Changing view acls to: rohitvg
17/03/25 15:35:12 INFO SecurityManager: Changing modify acls to: rohitvg
17/03/25 15:35:12 INFO SecurityManager: SecurityManager: authentication disabled; ui acls disabled; users with view permissions: Set(rohitvg); users with modify permissions: Set(rohitvg)
17/03/25 15:35:13 INFO Slf4jLogger: Slf4jLogger started
17/03/25 15:35:13 INFO Remoting: Starting remoting
17/03/25 15:35:13 INFO Remoting: Remoting started; listening on addresses :[akka.tcp://sparkDriver@192.168.0.13:65069]
17/03/25 15:35:13 INFO Utils: Successfully started service 'sparkDriver' on port 65069.
17/03/25 15:35:13 INFO SparkEnv: Registering MapOutputTracker
17/03/25 15:35:13 INFO SparkEnv: Registering BlockManagerMaster
17/03/25 15:35:13 INFO DiskBlockManager: Created local directory at /private/var/folders/qh/cgfy9d8j4933q486fyplbw0c0000gn/T/spark-2ea6005d-c56f-46f6-ba58-ab8d66fe6588/blockmgr-a5537a3e-99bc-4daa-a756-c8e3a87b98a6
17/03/25 15:35:13 INFO MemoryStore: MemoryStore started with capacity 983.1 MB
17/03/25 15:35:13 INFO HttpFileServer: HTTP File server directory is /private/var/folders/qh/cgfy9d8j4933q486fyplbw0c0000gn/T/spark-2ea6005d-c56f-46f6-ba58-ab8d66fe6588/httpd-0a3671d4-1365-440c-94e4-8c1f34315250
17/03/25 15:35:13 INFO HttpServer: Starting HTTP Server
17/03/25 15:35:13 INFO Utils: Successfully started service 'HTTP file server' on port 65070.
17/03/25 15:35:13 INFO SparkEnv: Registering OutputCommitCoordinator
17/03/25 15:35:13 INFO Utils: Successfully started service 'SparkUI' on port 4040.
17/03/25 15:35:13 INFO SparkUI: Started SparkUI at http://192.168.0.13:4040
17/03/25 15:35:13 INFO Executor: Starting executor ID driver on host localhost
17/03/25 15:35:14 INFO Utils: Successfully started service 'org.apache.spark.network.netty.NettyBlockTransferService' on port 65071.
17/03/25 15:35:14 INFO NettyBlockTransferService: Server created on 65071
17/03/25 15:35:14 INFO BlockManagerMaster: Trying to register BlockManager
17/03/25 15:35:14 INFO BlockManagerMasterEndpoint: Registering block manager localhost:65071 with 983.1 MB RAM, BlockManagerId(driver, localhost, 65071)
17/03/25 15:35:14 INFO BlockManagerMaster: Registered BlockManager
Eg. 1) Create a Pair RDD
(Organizer1,20000)
(Organizer1,10000)
(Organizer2,30000)
(Organizer1,20000)
(Organizer3,10000)
(Organizer1,20000)
(Organizer1,20000)
(Organizer2,10000)
(Organizer3,20000)
(Organizer1,20000)
Eg. 2) Group the Pair RDD using orgainzer as the key
(Organizer1,CompactBuffer(20000, 10000, 20000, 20000, 20000, 20000))
(Organizer3,CompactBuffer(10000, 20000))
(Organizer2,CompactBuffer(30000, 10000))
Eg. 3) Instead of grouping, reduce the pair rdd to organizer with the total budget
(Organizer1,110000)
(Organizer3,30000)
(Organizer2,40000)
Eg. 4) Instead of just reducing to organizer with the total of the budget, reduce to organizer and avg. budget

coupledValuesRdd: 
(Organizer1,(20000,1))
(Organizer1,(10000,1))
(Organizer2,(30000,1))
(Organizer1,(20000,1))
(Organizer3,(10000,1))
(Organizer1,(20000,1))
(Organizer1,(20000,1))
(Organizer2,(10000,1))
(Organizer3,(20000,1))
(Organizer1,(20000,1))

 intermediate : 
(Organizer1,(110000,6))
(Organizer3,(30000,2))
(Organizer2,(40000,2))

 averagesRdd : 
(Organizer1,18333)
(Organizer3,15000)
(Organizer2,20000)
```
