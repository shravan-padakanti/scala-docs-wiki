So far we focused on how transformations such as `map`, `flatMap`, `filter`, etc are distributed and parallelized on a cluster in Spark.

Here we will see how **actions** such as `fold`, `reduce` are **distributed** in Spark.

Operations like `fold`, `reduce` and `aggregate` have something in common: they walk through a collection and combine neighboring elements of the collections to produce a **single** combined result. Thus we call them **Reduction Operations**. Many of Spark's actions are reduction operations, but not all. E.g. Saving things to a file is an action which is executed eagerly, but its not a reduction operation.

### `foldLeft` vs `fold`

* foldLeft signature: 
    ```scala
    def foldLeft[B](z: B)(f: (B, A) => B): B
    ```
* fold signature: 
    ```scala
    def fold(z: A)(f: (A, A) => A): A
    ```

In the previous course, [we saw](https://github.com/rohitvg/scala-parallel-programming-3/wiki/Data-Parallel-Operations) that `fold` is parallelizable whereas `foldLeft` is not parallelizable since it passes the accumulator sequentially to fold in the left direction. Another example of why `foldLeft` is not parallelizable:

```scala
val xs = List(1,2,3,4)
val result - xs.foldLeft("")((str: String, i: Int) => str + i) // takes in a string accumulator, and combines it with an int to return a string..

// If we force parallelize this:
// List(1,2): "" + 1 = "1" + 2 = "12"
// List(3,4): "" + 3 = "3" + 4 = "34"
// Combination: ERROR: Type error - trying to combine String with String!
```

On the contrary, as seen in the signatures, `foldLeft` restricts us into returning/combining the same types. Hence it is parallelizable.


