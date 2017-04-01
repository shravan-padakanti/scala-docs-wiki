SQL is a common language used for doing analytics on databases. But it's a pain to connect big data processing pipelines like Spark or Hadoop to an SQL database.

We want to:
* seamlessly intermix SQL queries with Scala
* get all the optimizations used in databases on Spark Jobs

**Spark SQL delivers both these features!**

### Goals for Spark SQL

1. Support **relational processing** (SQL like syntax) both within Spark programs (on RDDs) and on external data sources with a friendly API. Sometimes its more desirable to express a computation in SQL syntax with functional APIs and vice versa.

2. **High performance**: achieved by using optimizations techniques used in databases.

3. Easily **support new data sources** such as semi-structured (like json) and structured data (like external databases), to get them easily into Spark.

## Spark SQL

**It is a component of the Spark Stack.**

* It is a Spark module for structured data processing.
* It is implemented as a library on top of Spark.

![spark_sql_in_spark_visual](https://github.com/rohitvg/scala-spark-4/blob/master/resources/images/spark_sql_in_spark_visual.png)

**Three main APIs that it provides:**

* SQL literal syntax
* `DataFrame`s
* `Dataset`s

**Two specialized backend components:**

* Catalyst: query optimizer
* Tungsten: off-heap serializer



