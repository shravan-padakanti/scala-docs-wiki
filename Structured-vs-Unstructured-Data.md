## Structured and Optimization

### Example

Lets say we are the CodeAward organization, and offer scholarships to programmers who have overcome adversity. We have the following 2 datasets:

```scala
case class Demographic( id: Int,
                        age: Int,
                        codingBootcamp: Boolean,
                        country: String,
                        gender: String,
                        isEthnicMinority: Boolean,
                        servedInMilitary: Boolean)
val demographics = sc.textfile(...)... // Pair RDD: (id, demographic)

case class Finances(id: Int,
                    hasDebt: Boolean,
                    hasFinancialDependents: Boolean,
                    hasStudentLoans: Boolean,
                    income: Int)
val finances = sc.textfile(...)... // Pair RDD: (id, finances)
```

Our goal is to tally up and select students for specific scholarship. For example, lets count:

* Swiss students
* With dept and financial dependents

How might we implement this Spark program?

**Possibility 1**

```scala
demographics.join(finances) // Pair RDD: (Int, (Demographic,Finances))
            .filter { p =>
                p._2._1.country == "Switzerland" &&
                p._2._2.hasFinancialDependents &&
                p._2._2.hasDebt
            }.count
```

1. Inner Join first
2. Filter to select people in Switzerland
3. Filter to select people with debt and financial dependents

**Possibility 2**

```scala
val filtered = finances.filter(p => p._2.hasFinancialDependents && p._2.hasDebt)

demographics.filter(p => p._2.country == "Switzerland")
            .join(filtered)
            .count
```

1. Filter down the database first (all people with debt and financial dependents)
2. Filter down people that have country as Switzerland
3. Inner Join on smaller, filtered down rdds

**Possibility 3**
 > Cartesian product of {A, K, Q, J, 10, 9, 8, 7, 6, 5, 4, 3, 2} with card suits {♠, ♥, ♦, ♣} gives 52 cards
```scala
val cartesian = demographics.cartesian(finances)

cartesian.filter {
        case (p1, p2) => p1._1 == p2._1
    }.filter {
        case (p1, p2) => (p1._2.country == ŏSwitzerlandŏ) &&
                         (p2._2.hasFinancialDependents) &&
                         (p2._2.hasDebt)
    }.count
```

1. Cartesian product on both rdds
2. Filter to select resulting of cartesian with same IDs
3. Filter to select people in Switzerland who have debt and financial dependents

### Example Analysis

While for all three of these possible solutions, **the end result is the same, the time it takes to execute the job is vastly different**.

Turns out, possibility 1 is 3.6 times slower than possibility 2, and possibility 3 is 177 times slower than possibility 2.

**Wouldn't it be nice if Spark automatically knew, if we wrote the code in possibility 3, that it could rewrite our code to possibility 2?**

**Given a bit of extra _structural information_, Spark can do many optimizations!**

## Structured and Unstructured Data

All data is not equal structurally. It falls on a spectrum from unstructured to structured.
![data_spectrum](https://github.com/rohitvg/scala-spark-4/blob/master/resources/images/data_spectrum.png)

### Structured Data Vs RDDs

Spark and RDDs don't know anything about the **schema** of the data it's dealing with. 

Given an arbitrary RDD, Spark knows that the RDD is parameterized with arbitrary types such as:

* Person
* Account
* Demographic
* etc

But it does not know anything about these types's structure.

Consider we have a case class Account:

```scala
case class Account(name: String, balance: Double, risk: Boolean)
```
and we have a RDD of of Account objects i.e. `RDD[Account]`:

Here **Spark/RDDs** see only Blobs of objects that are called Account. Spark cannot see inside this object or analyze how it may be used or optimized based on that usage. It's opaque.

![spark_sees](https://github.com/rohitvg/scala-spark-4/blob/master/resources/images/spark_sees.png)

On the other hand, a database/Hive performs computations on columns on named and typed values. So everything is know about the structure of the data and hence they are heavily optimized.

So if Spark could see data this way, it could break up the data and optimize on the available information.

The same can be said about **Computation**. 

In Spark:

* we do **functional transformations** on data.
* we pass user defined function literal to higher order functions like `map`,`flatMap`,`filter`.

Just like the data, the function literals are completely Opaque to spark as well. A user can do anything
inside of one of the above functions, and all Spark can see is something like: `$anon$1@604f1a67`

In Database/Hive: 

* we do **declarative transformations** on data.
* Specialized/structured, pre-defined operations.

Databases know the operations that are being done on the data. 