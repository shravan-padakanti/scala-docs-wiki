This is not a machine learning or data science course! It is about:

* distributed **data parallelism** in Spark.
* extending familiar functional abstractions like functional lists over large clusters.
* Context: analyzing large data sets.

Spark is a functionally oriented framework for large data set processing implemented in Scala.

### Why Scala and Spark?

For small datasets we can use **R/Python/Matlab**, but for large datasets where a single computer is not sufficient, these languages are unsuitable as they don't scale. To scale, we would have to manually figure out how to distribute the given dataset onto a computer cluster i.e distributed systems understanding is needed.

On the contrary, **Spark API is almost 1-to-1 with Scala**. Thus, by working in Scala, in a functional style, you can quickly scale your problem from one node to hundreds, or even thousands by leveraging
Spark, successful and performant large-scale data processing framework which looks a and feels a lot like Scala Collections!

### Hadoop vs Spark i.e Why Spark?

1. **Expressive:** Spark API is modeled after Scala collections. Look like functional lists! Richer, more composable operations possible than in Hadoop MapReduce which requires lot more code and boilerplate.
2. **More performant:** Spark is performant in terms of running time, and also in terms of developer productivity. This is because since its more fast, developers can be more Interactive as the state of the updated data is more easily visible.
3. **Good for data science:** Not just because of performance, but because it enables _iteration_ (multiple passes over same set of data), which is required by most algorithms in a data scientist’s toolbox. In Hadoop this is complex which needs extra libraries, extra map reduce, etc. On the contrary in Scala it is as simple as using a simple `for` loop.

### This Course

* Extending data parallel paradigm to the distributed case, using Spark.
* Spark’s programming model
* Distributing computation, and cluster topology in Spark
* How to improve performance; data locality, how to avoid re-computation and shuffles in Spark.
* Relational operations with DataFrames and Datasets

### Books

* Basics - O'Reilly - Learning Spark (2015), written by Holden Karau, Andy Konwinski, Patrick Wendell, and Matei Zaharia
* Basics: Spark in Action (2017), written by Petar Zecevic and Marko Bonaci
* Advanced: O'Reilly - High Performance Spark (in progress), written by Holden Karau and Rachel Warren
* Advanced: O'Reilly - Advanced Analytics with Spark (2015), written by Sandy Ryza, Uri Laserson, Sean Owen, and Josh Wills
* Deep advanced: https://www.gitbook.com/book/jaceklaskowski/mastering-apache-spark/details