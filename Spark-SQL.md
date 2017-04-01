SQL is a common language used for doing analytics on databases. But it's a pain to connect big data processing pipelines like Spark or Hadoop to an SQL database.

Everything about SQL is structured (fixed data-types, fixed set of operations). This rigidity has been used to get all kinds of performance speedups.

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

### Relational Queries (SQL)

Data is organized into one or more **tables**. Tables typically represent objects of a certain type, and they contain **columns** and **rows**.

Our terminology: A **relation** is just a table. Columns are **Attributes**. Rows are **records** or **tuples**.

### Spark SQL

**DataFrame** is Spark SQL's core abstraction, **conceptually equivalent to a table in a relational database**. Thus Dataframes are, conceptually, RDDs full of records with a **known schema**. (RDDs on the other hand do not have any schema info).

DataFrames are **untyped**. (unlike RDDs which have a type parameter (generic type parameter) : RDD[**T**]. Hence transformations on DataFrames are known as **untyped transformations**.

### SparkSession

`SparkSession` is the new `SparkContext`. We use it when we use Spark SQL.

```scala
import org.apache.spark.sql.SparkSession

val sparkSession = SparkSession.builder()
                               .appName("My App")
                               //.config("spark.some.config.option", "some-value")
                               .getOrCreate()
```

### Creating DataFrames

`DataFrame`s can be created in 2 ways:

1. From an existing RDD: Either with schema inference, or with an explicit schema
2. Reading in a specific data-source from a file: common structured or semi-structured formats such as JSON

### Examples 

#### 1a: From an existing RDD: with schema inference

Given a **Pair RDD**: `RDD[(T1, T2, ...., TN)]`, a `DataFrame` can be created with its schema automatically inferred using the `toDF` method:

```scala
val tupleRdd = ... // Assume RDD[(Int, String String, String)]
val tupleDF = tupleRdd.toDF("id", "name", "city", "country") // columnnames in the dataframe
```

**Note:** If `toDF` is used without arguments in the above case, then Spark assigns numbers as the attributes i.e. `_1, _2, _3, etc.` to the `DataFrame`.

If the RDD uses a type which is already a case class, then Spark can infer the attributes directly from the case class's fields:

```scala
case class Person(id: Int, name: String, city: String)
val peopleRDD = ... // Assume RDD[Person]
val peopleDF = peopleRDD.toDF
```

#### 1b: From an existing RDD: schema explicitly specified

It needs 3 steps:

1. create an RDD of `Rows` from the original RDD
2. create the schema represented by a `StructType` matching the structure of `Rows` in the RDD created in step 1.
3. Apply the schmea to the RDD of `Rows` via `createDataFrame` method provided by `SparkSession`

```scala
case class Person(name: String, age: Int)
val peopleRdd = sc.textFile(...) // Assume RDD[Person]

// The schema is encoded in a string
val schemaString = "name age"
// Generate the schema based on the string of schema
val fields = schemaString.split(",").map(fieldName => StructField(fieldName, StringType, nullable = true))
val schema = StructType(fields)

// Convert records of the RDD (people) to Rows
val rowRDD = peopleRDD.map(_.split(",")).map(attributes => Row(attributes(0), attributes(1).trim))
// Apply the schema to the RDD
val peopleDF = spark.createDataFrame(rowRDD, schema)
```

#### 2: By reading in a data source from file

Using the `SparkSession` object, you can read semi-structure/structured data by using the `read` method. 

**JSON, CSV, Parquet, JDBC** files can be directly read. Info on all methods avialble to do this: http://spark.apache.org/docs/latest/api/scala/#org.apache.spark.sql.DataFrameReader


Eg. for a JSON file:

```scala
val df = sparkSession.read.json("examples/src/main/resources/people.json")
```

## Spark SQL Literals

Once you have a `DataFrame`, you can now freely write familiar SQL syntax to operate on it!

Given a `dataframe`, we just have to register it as a temprary SQL view first. This essentially gives a name to our DataFrame in SQL so we can refer to it in an SQL FROM statement.

```scala
// Register the DataFrame as a SQL temporary view
peopleDF.createOrReplaceTempView("people")

// SQL literals can be passed to Spark SQLās sql method
val adultsDF = spark.sql("SELECT * FROM people WHERE age > 17")
```

The SQL Statements avaiable are largely what's available in **HiveQL**. This includes standard SQL statements such as:

* `SELECT`
* `FROM`
* `WHERE`
* `COUNT`
* `HAVING`
* `GROUP BY`
* `ORDER BY`
* `SORT BY`
* `DISTINCT`
* `JOIN`
* `(LEFT|RIGHT|FULL) OUTER JOIN`
* Subqueries: `SELECT col FROM ( SELECT a + b AS col from t1) t2`

**Supported Spark SQL Syntax:** 

* https://docs.datastax.com/en/datastax_enterprise/4.6/datastax_enterprise/spark/sparkSqlSupportedSyntax.html

**HiveQL CheatSheet:**

* https://hortonworks.com/blog/hive-cheat-sheet-for-sql-users/

**Updated list of supported Hive features in Spark SQL, the official Spark SQL docs enumerate:**

* https://spark.apache.org/docs/latest/sql-programming-guide.html#supported-hive-features

### An interesting SQL Query

Lets assume we have a `DataFrame` representing a data set of employees:

```scala
case class Employee(id: Int, fname: String, lname: String, age: Int, city: String)

// DataFrame with schema defined in Employee case class
val employeeDF = sc.parallelize(...).toDF

// employeeDF:
// +---+-----+-------+---+--------+
// | id|fname| lname |age| city   |
// +---+-----+-------+---+--------+
// | 12| Joe |  Smith| 38|New York|
// |563|Sally|  Owens| 48|New York|
// |645|Slate|Markham| 28|  Sydney|
// |221|David| Walker| 21|  Sydney|
// +---+-----+-------+---+--------+
```

Goal is to obtain just the IDs and Lastnames of employees working in a specific city, say Sydney, Australia. And we want to sort the result in order of imcreasing employee ID.

What would this SQL query look like?

```scala

val sydneyEmployeesDF = sparkSession.sql("""SELECT id, lname 
                                            FROM employees 
                                            WHERE city = "sydney"
                                            ORDER BY id""")
// sydneyEmployeesDF:
// +---+-------+
// | id|  lname|
// +---+-------+
// |221| Walker|
// |645|Markham|
// +---+-------+
```

**Note:** Its best to use Spark 2.1+ with Scala 2.11+ for doing SQL queries with Spark SQL.