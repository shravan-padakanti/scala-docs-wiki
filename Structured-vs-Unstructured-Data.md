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

**Given a bit of extra structureal information, Spark can do many optimizations!**

