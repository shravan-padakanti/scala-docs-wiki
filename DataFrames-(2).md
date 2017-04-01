## Cleaning Data with DataFrames

Sometimes the dataset has `null` or `NaN` values. In these cases, we want to:

* drop rows/records with these unwanted values
* replace unwanted values with a constant

#### Drop rows/records with unwanted values 

* `drop()`: Drops rows containing `null` or `NaN` value in **any** column, and returns a new DataFrame.
* `drop("all")`: Drops rows containing `null` or `NaN` value in **all** columns, and returns a new DataFrame
* `drop(Array("id","name"))`: Drops rows containing `null` or `NaN` value in **specified** columns, and returns a new DataFrame

#### Replacing unwanted values with a constant

* `fill(0)`: Replaces all occurrences of `null` or `NaN` value in **numeric** columns with the specified value, and returns a new DataFrame.
* `fill(Map("minBalance" -> 0))`: Replaces all occurrences of `null` or `NaN` value in the **specified** column with the specified value, and returns a new DataFrame.
* `replace(Array("id"),Map(1234 -> 8923))`: Replaces **specified value** (1234) in the **specified column** (id) with the **specified value** (8923), and returns a new DataFrame.

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

### Revisiting previous example for Join
