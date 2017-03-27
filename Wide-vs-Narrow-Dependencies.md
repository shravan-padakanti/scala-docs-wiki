In this session, we are going to focus on wide versus narrow dependencies, which dictate relationships between RDDs in graphs of computation, which we'll see has a lot to do with shuffling. 

So far, we have seen that **some transformations significantly more expensive (latency) than others**. 

In this session we will: 

* look at how RDD's are represented
* dive into how and when Spark decides it must shuffle data
* see how these dependencies make **fault tolerance** possible

## Lineages

Computations on RDDs are represented as a **lineage graph**, a DAG representing the computations done on the RDD. This representation/DAG is what Spark analyzes to do optimizations. Because of this, for a particular operation, it is possible to step back and figure out how a result of a computation is derived from a particular point.

E.g.: 

```scala
val rdd = sc.textFile(...)
val filtered = rdd.map(...).filter(...).persist()
val count = filtered.count()
val reduced = filtered.reduce()
```
![lineage_graph](https://github.com/rohitvg/scala-spark-4/blob/master/resources/images/lineage_graph.png)

## How RDDs are represented

RDDs are made up of 4 parts: 

* **Partitions**: Atomic pieces of the dataset. One or many per compute node.
* **Dependencies**: Models relationship between this RDD and **its partitions** with the RDD(s) it was derived from. (Note that the dependencies maybe modeled per partition as shown below). 
* A **function** for computing the dataset based on its parent RDDs.
* **Metadata** about it partitioning scheme and data placement.

![rdd_anatomy_1](https://github.com/rohitvg/scala-spark-4/blob/master/resources/images/rdd_anatomy_1.png) ![rdd_anatomy_2](https://github.com/rohitvg/scala-spark-4/blob/master/resources/images/rdd_anatomy_2.png)

## RDD Dependencies and Shuffles

[Previously](https://github.com/rohitvg/scala-spark-4/wiki/Optimizing-with-Partitioners#how-do-i-know-when-a-shuffle-will-occur) we saw the **Rule of thumb**: a shuffle can occur when the resulting RDD depends on other elements from the same RDD or another RDD.

**In fact, RDD dependencies encode when data must move across network.** Thus they tell us when data is going to be shuffled.

Transformations cause shuffles, and can have 2 kinds of dependencies:

1. Narrow dependencies: Each partition of the parent RDD is used by at most one partition of the child RDD. 
    ```
    [parent RDD partition] ---> [child RDD partition]
    ```
    **Fast!** No shuffle necessary. Optimizations like pipelining possible. Thus transformations which have narrow dependencies are fast.
1. Wide dependencies: Each partition of the parent RDD may be used by multiple child partitions
    ```
                           ---> [child RDD partition 1]
    [parent RDD partition] ---> [child RDD partition 2]
                           ---> [child RDD partition 3]
    ```
    **Slow!** Shuffle necessary for all or some data over the network. Thus transformations which have narrow dependencies are slow.

### Visual: Narrow dependencies Vs. Wide dependencies




