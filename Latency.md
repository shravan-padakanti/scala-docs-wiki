

In the Parallel Programming course we learned about:

* Data Parallelism in the single machine, multi-core, multiprocessor world.
* _Parallel Collections_ as an implementation of this paradigm.

Here we will learn:

* Data Parallelism in a distributed (multi node) setting
* _Distributed collections_ abstraction from Apache Spark as an implementation of this paradigm.

Because of the **distribution**, we have 2 _new_ issues:

1. Partial Failure: crash failures on a subset of machines in the cluster.
2. Latency: network communication causes higher latency in some operations - **cannot be masked and always present; impacts programming model as well as code directly as we try to reduce network communication**. 

**Apache Spark** stands out in the way it handles these issues.


