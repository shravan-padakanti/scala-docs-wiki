In this session we're going to talk about the anatomy of a Spark job. We're going to look at how clusters are typically organized that Spark runs on. And this is actually important. It's going to come back to the programming model once again. You can't just pretend like you have sequential collections that are on one machine. You actually have to think about how your program might be spread out along the cluster. 

## Spark Jobs anatomy

Spark jobs are organized in **Master - Worker** topology. Usually there is **1 Master, many Workers**. The node that acts as the Master is called as the **Driver Program** and holds the `SparkContext`, thus this is the node that we i.e. our program interacts with. The Workers nodes are called as **Executors** and these execute the Jobs.

How do the Master and the Workers communicate? They do this via a **Cluster Manager** (e.g. YARN, Mesos) which manages/allocates resources, scheduling, etc.

![spark_anatomy](https://github.com/rohitvg/scala-spark-4/blob/master/resources/images/spark_anatomy.png)

Thus, the **Spark application is a set of processes running on a cluster**. 

The Driver-Program:

* coordinates all the processes.
* holds the process where the `main()` method of our program runs.
* holds the process that creates `SparkContext`, creates `RDD`s, and stages up or sends off transformations and actions.

The executors:

* run the task which represent the application
* return computed results to the driver
* provide in-memory storage for cached `RDD`s.

### How Spark Jobs are executed

1. The driver-program runs the Spark application, which creates a `SparkContext` upon start-up.
2. The `SparkContext` connects to a **cluster-manager** which allocates resources.
3. Spark acquires executors on nodes in the cluster.
4. Driver-program sends your application code to the executors.
5. Finally, `SparkContext` sends tasks for the executors to run.

### Example 1: `println`

Assume we have an `RDD` populated with `Person` objects.

```scala
case class Person(name: String, age: Int)

val people: RDD[Person] = ...
people.foreach(println)
```

What happens?

* Nothing happens on the driver. This is because `foreach` is an action with `Unit` return type. Hence it is eagerly executed on the executors, not the driver. Thus any calls to `println` are visible only on `stdout` of worker nodes and not the master node.

### Example 2: `take`

```scala
case class Person(name: String, age: Int)

val people: RDD[Person] = ...
val first10 = people.take(10)
```

What happens? Where will the `Array[Person]` representing `first10` end up?

It ends of on the driver program. In general, executing an action involves communication between worker
nodes and the node running the driver program (since its an action, the workers perform the action, and send result to the driver where it is aggregated.).

**Moral of the story:** To make effective user of RDDs, you have to understand little bit about how Spark works. As due to the lazy/eager properties, it is not obvious upon first glance that on what part of the cluster a line of code might run on.




