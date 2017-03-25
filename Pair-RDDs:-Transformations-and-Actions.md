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

### `groupByKey`

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

### `reduceByKey`

It is a combination of `groupByKey` followed by `reduce` on values of each grouped collection. It is more efficient than using the both separately.

``scala
def reduceByKey( func(V, V) => V ): RDD[(K, V)] // V corresponds to the values of Pair RDD, we only operate on the value since a pair RDD is in the form of Key Values.

/* Eg. Going back to the last events example, if we want to calculate the total budget per organization, then: */
val eventsRdd = rdd.map(event => (event.organizer, event.budget)) // Pair RDD
val totalBudgets = eventsRdd.reduceByKey( _ + _ ) // at this point we already have keys and values. So reduceByKey means reduce the values corresponding to the given key using the given function
// (Organizer1, 42000)
// (Organizer2, 151400)
```
