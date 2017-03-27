In this session, we are going to focus on wide versus narrow dependencies, which dictate relationships between RDDs in graphs of computation, which we'll see has a lot to do with shuffling. 

So far, we have seen that **some transformations significantly more expensive (latency) than others**. 

In this session we will: 

* look at how RDD's are represented
* dive into how and when Spark decides it must shuffle data
* see how these dependencies make **fault tolerance** possible

## Terminology

### Lineages

Computations on RDDs are represented as a **lineage graph**, a DAG representing the computations done on the RDD. This representation/DAG is what Spark analyzes to do optimizations.

E.g.: 

```scala
val rdd = sc.textFile(...)
val filtered = rdd.map(...).filter(...).persist()
val count = filtered.count()
val reduced = filtered.reduce()
```
![lineage_graph]



