So far we focused on how transformations such as `map`, `flatMap`, `filter`, etc are distributed and parallelized on a cluster in Spark.

Here we will see how **actions** such as `fold`, `reduce` are **distributed** in Spark.

Operations like `fold`, `reduce` and `aggregate` have something in common: they walk through a collection and combine neighboring elements of the collections to produce a **single** combined result. Thus we call them **Reduction Operations**. Many of Spark's actions are reduction operations, but not all. E.g. Saving things to a file is an action which is executed eagerly, but its not a reduction operation.

In the previous course, [we saw](https://github.com/rohitvg/scala-parallel-programming-3/wiki/Data-Parallel-Operations) that `fold` is parallelizable whereas `foldLeft` is not parallelizable since it passes the accumulator sequentially to fold in the left direction. 


