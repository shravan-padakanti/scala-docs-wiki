## Cleaning Data with DataFrames

Sometimes the dataset has `null` or `NaN` values. In these cases, we want to:

* drop rows/records with these unwanted values
* replace unwanted values with a constant

#### Drop rows/records with unwanted values 

* `drop()`: Drops rows containing `null` or `NaN` value in **any** of the column of that row, and returns a new DataFrame.
* `drop("all")`: Drops rows only if they contain `null` or `NaN` value in **all** the columns of that row, and returns a new DataFrame
* `drop(Array("colname1","colname2"))`: Drops rows containing `null` or `NaN` value in the **specified** columns, and returns a new DataFrame

#### Replacing unwanted values with a constant

* `fill(0)`: Replaces all occurrences of `null` or `NaN` value in **numeric** columns with the specified value (0 in this case), and returns a new DataFrame.
* `fill(Map("colname1" -> 0))`: Replaces all occurrences of `null` or `NaN` value in the **specified** column with the specified value, and returns a new DataFrame.
* `replace(Array("colname1"),Map(1234 -> 8923))`: Replaces **specified value** (1234) in the **specified column** (colname1) with the **specified value** (8923), and returns a new DataFrame.

## Common Actions on DataFrame

Like RDDs, DatFrames also have their own set of **actions**, some of which we have already seen in the last lecture.

```scala
collect(): Array[Row]
// Returns an a\Array consisting of all the Rows in the DatFrame

count(): Long
// Returns the no.of rows in the DataFrame

first(): Row/head(): Row
// Returns the first row in the DataFrame

show(): Unit
// Displays the top 20 rows of the DataFrame in a tabular form 

take(n: Int): Array[Row]
//Returns the first n rows of the DataFrame
```

## Joins on DataFrames

These are similar to the Joins on Pair RDDs [that we have previously seen](https://github.com/rohitvg/scala-spark-4/wiki/Joins), with **one major difference**: since DataFrames are not Key/Value pairs, we have to specify which columns we have to join on.

We have the following join types available:

* `inner`
* `outer`
* `left_outer`
* `right_outer`
* `leftsemi`

#### Performing Joins

Given 2 DataFrames df1, df2, each having a column/attribute called `id`, we can do an inner join as follows:

```scala
df1.innerJoin(df2, $"df1.id" === $"df2.id")
```

Its possible to change the join type by passing an additional string parameter to `join` specifying which type of join to perform:

```scala
df1.innerJoin(df2, $"df1.id" === $"df2.id", "right_outer")
```

So, as seen, here we have to specify the type of join as a string argument unlike Pair RDDs where we had separate join methods.

### Revisiting previous example for Join

[Recall over previous example](https://github.com/rohitvg/scala-spark-4/wiki/Pair-RDDs:-Joins#inner-join-join):

Here we convert the Pair RDDs to DataFrames using case classes inside the lists and the `toDF` function on them.

```scala
val sc = SparkHelper.sc
val sqlContext = new SQLContext(sc)
import sqlContext.implicits._

case class Subscription(id: Int, v: (String, String))
case class Location(id: Int, v: String)

val as = List(Subscription(101, ("Hanson", "Bart")),
              Subscription(102, ("Thomas", "Clipper")),
              Subscription(103, ("John", "ClipperVisa")),
              Subscription(104, ("Chu", "Clipper")))
val subscriptions = sc.parallelize(as) 
val subscriptionsDF = subscriptions.toDF.show()

val ls = List(Location(101, "Chicago"),
              Location(101, "SanFranciso"),
              Location(102, "SantaClara"),
              Location(102, "SanJose"),
              Location(103, "MountainView"),
              Location(103, "Monterey"))
val locations = sc.parallelize(ls)
val locationsDF = locations.toDF.show()

// subscriptionsDF
//  +---+------------------+
//  | id|                 v|
//  +---+------------------+
//  |101|     [Hanson,Bart]|
//  |102|  [Thomas,Clipper]|
//  |103|[John,ClipperVisa]|
//  |104|     [Chu,Clipper]|
//  +---+------------------+
//  
// locationsDF
//  +---+------------+
//  | id|           v|
//  +---+------------+
//  |101|     Chicago|
//  |101| SanFranciso|
//  |102|  SantaClara|
//  |102|     SanJose|
//  |103|MountainView|
//  |103|    Monterey|
//  +---+------------+

```

How do we combine only customers that have a subscription and have location info?

We perform an inner join: 

```scala
val trackedCustomersDF = subscriptionsDF.join(locationsDF, subscriptionsDF("id") === locationsDF("id"))

// +---+------------------+---+------------+
// | id|                 v| id|           v|
// +---+------------------+---+------------+
// |101|     [Hanson,Bart]|101|     Chicago|
// |101|     [Hanson,Bart]|101| SanFranciso|
// |103|[John,ClipperVisa]|103|MountainView|
// |103|[John,ClipperVisa]|103|    Monterey|
// |102|  [Thomas,Clipper]|102|  SantaClara|
// |102|  [Thomas,Clipper]|102|     SanJose|
// +---+------------------+---+------------+
```
**As expected, customer 104 is missing as inner join is not loss-less!**

Now, we want to know for which subscribers we have location information. E.g. it is possible that someone has a subscription but uses cash for tickets, hence doesn't show in location info. Like customer 104. 

```scala
val subscriptionsWithOptionalLocationInfoDF = subscriptionsDF.join(locationsDF, subscriptionsDF("id") === locationsDF("id"), "left_outer")
subscriptionsWithOptionalLocationInfoDF.show()

// +---+------------------+----+------------+
// | id|                 v|  id|           v|
// +---+------------------+----+------------+
// |101|     [Hanson,Bart]| 101|     Chicago|
// |101|     [Hanson,Bart]| 101| SanFranciso|
// |103|[John,ClipperVisa]| 103|MountainView|
// |103|[John,ClipperVisa]| 103|    Monterey|
// |102|  [Thomas,Clipper]| 102|  SantaClara|
// |102|  [Thomas,Clipper]| 102|     SanJose|
// |104|     [Chu,Clipper]|null|        null|
// +---+------------------+----+------------+
```
Now we can do a filter on the right columns with value `null` to find the exact customers with no location info, and suggest them to download the app.

### Revisiting Scholarship Recipients example

[Lets revisit another example that we have seen](https://github.com/rohitvg/scala-spark-4/wiki/Structured-vs-Unstructured-Data#example).

```scala
case class Demographic( id: Int,
                        age: Int,
                        codingBootcamp: Boolean,
                        country: String,
                        gender: String,
                        isEthnicMinority: Boolean,
                        servedInMilitary: Boolean)
val demographicsDF = sc.textfile(...).toDF

case class Finances(id: Int,
                    hasDebt: Boolean,
                    hasFinancialDependents: Boolean,
                    hasStudentLoans: Boolean,
                    income: Int)
val financesDF = sc.textfile(...).toDF // Pair RDD: (id, finances)
```
As then, our goal is to tally up and select students for specific scholarship. For example, lets count:

* Swiss students
* With dept and financial dependents

We will do this using a DataFrame API this time:

```scala
demographicsDF.join(financesDF, demographicsDF("ID") === financesDF("ID"), "inner")
              .filter($"HasDebt" && $"HasFinancialDependents")
              .filter($"CountryLive" === "Switzerland")
              .count
```

In Practical tests, the DataFrame solution is the fastest than the any of the handwritten solutions we saw!

**How is this possible?**

Spark comes with 2 specialized backend components:

* Catalyst: Query optimizer.
* Tungsten: Off heap serializer.

Let's briefly develop some intuition about why structured data and computation enable these components to do so many optimizations.

#### Catalyst

Remember that Spark SQL sits on top of the Spark framework:

![spark_sql_in_spark_visual]https://github.com/rohitvg/scala-spark-4/raw/master/resources/images/spark_sql_in_spark_visual.png

It takes in user programs with this relational DataFrame API. So users can write relational code. Ultimately it spits out highly optimized RDDs that are run on regular Spark. And these highly optimized RDDs are arrived at by this Catalyst optimizer. So on the one hand you put in relational operations. Catalyst optimizer runs, and then we get out on the other end RDDs. 

So bottomline is **Catalyst compiles Spark SQL programs down to an optimized RDD.**

Assuming Catalyst...

* has full knowledge and understanding of all data types used in our program
* knows the exact schema of our data
* has detailed knowledge of the computations we would like to do

It does optimizations like:

* Reordering operations: Laziness + structure give the ability to analyze and rearrange DAG of computation before execution.
* Reduce the amount of data that is read: by not moving the unneeded data around the network.
* Pruning unneeded partitioning: skip partitions that are not needed in our computations

#### Tungsten

Since the data-types are restricted to Spark SQL types, Tungsten can provide:

* highly specialized data encoders: Tungsten can tightly pack serialized data into memory based on the schema information. Thus more data fits in memory and faster serialization/de-serialization.
* column based: Most operations done on tables tend to be focused on specific columns/attributes of the dataset. Thus, when storing data, groups data by column instead of row for faster lookups of data associated with specific attributes/columns. 
* off-heap (free from garbage collection overhead!)

Taken together, Catalyst and Tungsten offer ways to significantly speed up the code, even if it is written inefficiently.