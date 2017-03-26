As seen in the previous lecture, while **shuffling** Spark uses **hash partitioning** to determine which key-value pair should be sent to which machine.

The data within an RDD is split into several partitions:

Properties of partitions:

* Partitions never span multiple machines, i.e. data in the same partition is always guaranteed to be on the same machine.
* Each machine in the cluster contains one or more partitions.
* The **number of partitions** to use is **configurable**. By default, it equals **total number of cores on all executor nodes**. Eg. 6 worker nodes, with 4 cores each would have 24 partitions in the RDD.

Two kinds of partitioning available in Spark: 

* Hash partitioning
* Range partitioning

**Customizing a partitioning is only possible on Pair RDDs**.

### Hash Partitioning

Given a Pair RDD that should be grouped:

```scala
val purchasesPerMonth = purchasesRdd.map( p => (p.customerId, p.price) ) // pair RDD
                                    .groupByKey()                        // RDD[K, Iterable[V]] i.e RDD[p.customerId, Iterable[p.price]]
```
`groupByKey` first computes per tuple `(k,v)` its partition `p`:

```
p = k.hashCode() % numPartitions
```

Then, all tuples in the same partition `p` are sent to the machine hosting `p`.

**Intuition:** hash partitioning attempts to spread data evenly across partitions _based on the key_.

### Range Partitioning

Pair RDDs may contain keys that may have an _ordering_ defined. Eg. If the key is `Int`, `Char`, `String` ...

For such RDDs, **range partitioning** may be more efficient.

Using a range partitioner, keys are partitioned according to 2 things:

1. an _ordering_ for keys
2. a set of _sorted ranges_ of keys

**Property**: tuples with keys in the same range appear on the same machine.

### Example: Hash Partitioning

Consider a Pair RDD, with keys: `[8, 96, 240, 400, 401, 800]`, and a desired number of partitions of 4.

We know 

```
p = k.hashCode() % numPartitions
  = k % 4
``` 
Thus, based on this the keys are distributed as follows:

* Partition 0: `[8, 96, 240, 400, 800]`
* Partition 1: `[401]`
* Partition 2: `[]`
* Partition 3: `[]`

The result is a very unbalanced distribution which hurts performance, since the data is spread mostly on 1 node, so not very parallel.

### Example: Range Partitioning

This can improve the distribution significantly

* Assumptions: (a) Keys non-negative, (b) 800 is the biggest key in the RDD
* Set of ranges: from (800/4): `[1-200], [201-400], [401-600], [601-800]`

Thus, based on this the keys are distributed as follows:

* Partition 0: `[8, 96]`
* Partition 1: `[240, 400]`
* Partition 2: `[401]`
* Partition 3: `[800]`

This is much more balanced.

### Customizing Partitioning Data

