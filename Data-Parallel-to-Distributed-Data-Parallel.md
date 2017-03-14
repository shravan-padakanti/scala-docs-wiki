In this session, we're going to try and bridge the gap between **data parallelism in the shared memory case** (which is what we learned in the [parallel programming](https://github.com/rohitvg/scala-parallel-programming-3/wiki/Data-Parallel-Programming) course) and **distributed data parallelism**. So we will taking that idea of data parallelism and extending that to the situation where you no longer have data on just one node anymore. Now you have data spread across several independent nodes. 

What does data-parallelism look like?

Lets say we have a dataset which is in form of a single colleciton. We want to process this collection in a parallel fashion in order to speed it up. 
Our collection is a jar of jelly beans, called `jar'.

```scala
val res = jar.map(jellybean => doSomething(jellybean))
```
using **parallel collections** in Scala. 

> Note that it doesn't matter if we're doing this in parallel collections or sequential collections. We usually have the same API available to us.

## Visualizing "Shared Memory" Data Parallelism

Since we are talking about **Shared Memory** case, we assume we have only one node, one computer to process and this is what happens under the hood:

* Split the data
* Workers/threads independently operate on the split data in parallel
* Combine when done

**Scala's Parallel Collections is a collections abstraction over Shared Memory data-parallel execution**.

## Visualizing "Distributed" Data Parallelism

Here we have multiple computers i.e independednt nodes to process, and this is what happens under the hood:

* Split the data **over several nodes**.
* **Nodes** independently operate on the split data in parallel
* Combine when done

Here we have a new concern, that we did not have is the network, was the **latency** involved with these nodes having to share data or to communicate with one another in some way. This impacts the programming model. However, just like in parallel collections, **we can still keep the same familiar Scala collections abstraction over our distributed data-parallel execution**. So the code is the same. And in this case, a jar can be one of these distributed collections and spark. And yet we have this wonderful collections API that looks just like Scala collections, over top of this distributed data-parallel execution.

## Summary: Shared memory vs Distributed data-parallelism

[picture]

**Shared Memory Case:** Data-parallel programming model. Data partitioned in memory and operated upon in parallel.

**Distributed Case:** Data-parallel programming mode. Data partitioned between machines, network in between, operated upon in parallel. 

So most concepts and ideas from the Shared Memory case are applicable in the Distributed case, but here we have to take **latency** into consideration.

**Note:** Spark stands out the way it handles the latency issue.

## Apache Spark:

We use the **Apache Spark** framework for **distributed data-parallel programming**.

Spark implements a **distributed data-parallel model** called **Resilient Distributed Datasets (RDDs)**. It is the distributed counterpart of a parallel collection. 
I.e. When spark gets a huge dataset, it splits it up using some partiontioning mechanism and distributes this partitioned dataset across a cluster nodes and returns a reference to this entire distributed data in the form of a RDD. As mentioned, his is nothing but a distributed counterpart of a parallel collection, so we can use our familiar operations on it like:

```scala
val wiki: RDD[WikiArticle] = ...
wiki.map(article => article.toText.toLowerCase)
```
Thus it looks just like Scala collections, except that it's going to be distributed over many nodes. 