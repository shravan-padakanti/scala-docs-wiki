Last lecture, we learned about the **hash partitioners** and **range partitioners**, and also learned about what kind of operations may introduce new partitioners or which may discard custom partitioners.

### Why would we want to repartition the data?

It can bring substantial performance gains, especially in face of shuffles.

[We saw](https://github.com/rohitvg/scala-spark-4/wiki/Shuffling:-What-it-is-and-why-it's-important#example) how using `reduceByKey` instead of `groupByKey` localizes data better due to different partitioning strategy and thus reduces latency to deliver performance gains.
