Joins are another sort of **transformation** on **Pair RDDs** which are used to combine 2 Pair RDDs into 1 Pair RDD. There are 2 types:

1. Inner Joins (`join`)
2. Outer Joins (`leftOuterJoin`/`rightOuterJoin`)

The difference between them is what happens to the keys when RDDs don't contain the same key.

### Inner Join `join`

This takes in to Pair RDDs, and returns a single Pair Rdd whose **keys are present in both input RDDs**. Thus its a *lossy* transformation.

```scala
def join[W](other: RDD[(K, V)]): RDD[(K, (V, W))]
```

Example:

```scala
// List containing (customer_id, (last_name, subsription_card_name))
val as = List((101, ("Hanson", "Bart")), (102, ("Thomas", "Clipper")), (103, ("John", "ClipperVisa")),(104, ("Chu", "Clipper")))
val subscriptions = sc.parallelize(as) // Pair Rdd with key = customer_Id, value = (last_name, subsription_card_name)

// List containing (customer_id, most_visited_city)
val ls = List((101, "Chicago"), (101, "SanFranciso"), (102, "SantaClara"), (102, "SanJose"), (103, "MountainView"), (103, "Monterey"))
val locations = sc.parallelize(ls)  // Pair Rdd with key = customer_Id, value = most_visited_city

// To find that have a subscription as well as location info, we can call inner join:
val innerJoinedRdd = subscriptions.join(locations)
println("subscriptions.join(locations)")
innerJoinedRdd.collect().foreach(println)

// NOTE: we could have also called locations.join(subscriptions) for the same result
val innerJoinedRdd2 = locations.join(subscriptions)
println("locations.join(subscriptions)")
innerJoinedRdd2.collect().foreach(println)
```
Output:
```
Using Spark's default log4j profile: org/apache/spark/log4j-defaults.properties
17/03/25 16:50:40 INFO SparkContext: Running Spark version 1.4.0
17/03/25 16:50:40 WARN NativeCodeLoader: Unable to load native-hadoop library for your platform... using builtin-java classes where applicable
17/03/25 16:50:45 INFO SecurityManager: Changing view acls to: rohitvg
17/03/25 16:50:45 INFO SecurityManager: Changing modify acls to: rohitvg
17/03/25 16:50:45 INFO SecurityManager: SecurityManager: authentication disabled; ui acls disabled; users with view permissions: Set(rohitvg); users with modify permissions: Set(rohitvg)
17/03/25 16:50:46 INFO Slf4jLogger: Slf4jLogger started
17/03/25 16:50:46 INFO Remoting: Starting remoting
17/03/25 16:50:46 INFO Remoting: Remoting started; listening on addresses :[akka.tcp://sparkDriver@192.168.0.13:49192]
17/03/25 16:50:46 INFO Utils: Successfully started service 'sparkDriver' on port 49192.
17/03/25 16:50:46 INFO SparkEnv: Registering MapOutputTracker
17/03/25 16:50:46 INFO SparkEnv: Registering BlockManagerMaster
17/03/25 16:50:46 INFO DiskBlockManager: Created local directory at /private/var/folders/qh/cgfy9d8j4933q486fyplbw0c0000gn/T/spark-316f459e-18ee-41c2-aabf-4a31c510a05e/blockmgr-d1343105-5858-4d8a-89c2-01a267784aa4
17/03/25 16:50:46 INFO MemoryStore: MemoryStore started with capacity 983.1 MB
17/03/25 16:50:46 INFO HttpFileServer: HTTP File server directory is /private/var/folders/qh/cgfy9d8j4933q486fyplbw0c0000gn/T/spark-316f459e-18ee-41c2-aabf-4a31c510a05e/httpd-c97fe570-1233-4510-b27c-15d34511d1e6
17/03/25 16:50:46 INFO HttpServer: Starting HTTP Server
17/03/25 16:50:46 INFO Utils: Successfully started service 'HTTP file server' on port 49193.
17/03/25 16:50:46 INFO SparkEnv: Registering OutputCommitCoordinator
17/03/25 16:50:47 INFO Utils: Successfully started service 'SparkUI' on port 4040.
17/03/25 16:50:47 INFO SparkUI: Started SparkUI at http://192.168.0.13:4040
17/03/25 16:50:47 INFO Executor: Starting executor ID driver on host localhost
17/03/25 16:50:47 INFO Utils: Successfully started service 'org.apache.spark.network.netty.NettyBlockTransferService' on port 49194.
17/03/25 16:50:47 INFO NettyBlockTransferService: Server created on 49194
17/03/25 16:50:47 INFO BlockManagerMaster: Trying to register BlockManager
17/03/25 16:50:47 INFO BlockManagerMasterEndpoint: Registering block manager localhost:49194 with 983.1 MB RAM, BlockManagerId(driver, localhost, 49194)
17/03/25 16:50:47 INFO BlockManagerMaster: Registered BlockManager
Inner join: 

 subscriptions.join(locations): 

(101,((Hanson,Bart),Chicago))
(101,((Hanson,Bart),SanFranciso))
(102,((Thomas,Clipper),SantaClara))
(102,((Thomas,Clipper),SanJose))
(103,((John,ClipperVisa),MountainView))
(103,((John,ClipperVisa),Monterey))

 locations.join(subscriptions): 

(101,(Chicago,(Hanson,Bart)))
(101,(SanFranciso,(Hanson,Bart)))
(102,(SantaClara,(Thomas,Clipper)))
(102,(SanJose,(Thomas,Clipper)))
(103,(MountainView,(John,ClipperVisa)))
(103,(Monterey,(John,ClipperVisa)))
```