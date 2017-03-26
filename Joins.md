Joins are another sort of **transformation** on **Pair RDDs** which are used to combine 2 Pair RDDs into 1 Pair RDD. There are 2 types:

1. Inner Joins (`join`)
2. Outer Joins (`leftOuterJoin`/`rightOuterJoin`)

The difference between them is what happens to the keys when RDDs don't contain the same key.

### Inner Join `join`

This takes in 2 Pair RDDs, and returns 1 Pair Rdd whose **keys are present in both input RDDs**. Thus its a *lossy* transformation.

```scala
def join[W](other: RDD[(K, V)]): RDD[(K, (V, W))]
```

Example:

```scala
// List containing (customer_id, (last_name, subsription_card_name))
val as = List((101, ("Hanson", "Bart")), (102, ("Thomas", "Clipper")), (103, ("John", "ClipperVisa")),(104, ("Chu", "Clipper")))
val subscriptions = sc.parallelize(as) // Pair Rdd with key = customer_Id, value = (last_name, subsription_card_name)

// List containing (customer_id, most_visited_city). Contains all customer who use cards and thus can be tracked.
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

### Outer Joins (`leftOuterJoin`/`rightOuterJoin`)

This takes in 2 Pair RDDs, and returns 1 Pair Rdd whose **keys don't have to be present in both input RDDs**. Thus its a *lossless* transformation.

```scala
// to keep all the left rdd keys (Option can be none)
def leftOuterJoin[W](other: RDD[(K, W)]): RDD[(K, (V, Option[W]))] 
// to keep all the right rdd keys (Option can be none)
def rightOuterJoin[W](other: RDD[(K, W)]): RDD[(K, (Option[V], W))]
```

Example:

To find a list of all the customers who have subscriptions, including ones that don't exist in the locations list as they pay using cash and hence cannot be tracked:
```scala
// List containing (customer_id, (last_name, subsription_card_name))
val as = List((101, ("Hanson", "Bart")), (102, ("Thomas", "Clipper")), (103, ("John", "ClipperVisa")),(104, ("Chu", "Clipper")))
val subscriptions = sc.parallelize(as) // Pair Rdd with key = customer_Id, value = (last_name, subsription_card_name)

// List containing (customer_id, most_visited_city). Contains all customer who use cards and thus can be tracked.
val ls = List((101, "Chicago"), (101, "SanFranciso"), (102, "SantaClara"), (102, "SanJose"), (103, "MountainView"), (103, "Monterey"))
val locations = sc.parallelize(ls)  // Pair Rdd with key = customer_Id, value = most_visited_city

// Here we have to call the leftOuterJoin, as we need all the customers who have subscriptions. The second element in the combination i.e. the value from the second list can be null, which is okay for our requirement.
val leftOuterJoinedRdd = subscriptions.leftOuterJoin(locations)
println("subscriptions.leftOuterJoin(locations)")
leftOuterJoinedRdd.collect().foreach(println)
```
Output:
```
subscriptions.leftOuterJoin(locations)
(104,((Chu,Clipper),None))                       // <-- second value element is None
(101,((Hanson,Bart),Some(Chicago)))
(101,((Hanson,Bart),Some(SanFranciso)))
(102,((Thomas,Clipper),Some(SantaClara)))
(102,((Thomas,Clipper),Some(SanJose)))
(103,((John,ClipperVisa),Some(MountainView)))
(103,((John,ClipperVisa),Some(Monterey)))
```
