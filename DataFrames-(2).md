## Cleaning Data with DataFrames

Sometimes the dataset has `null` or `NaN` values. In these cases, we want to:

* drop rows/records with these unwanted values
* replace unwanted values with a constant

### Drop rows/records with unwanted values 

* `drop()`: Drops rows containing `null` or `NaN` value in **any** column, and returns a new DataFrame.
* `drop("all")`: Drops rows containing `null` or `NaN` value in **all** columns, and returns a new DataFrame
* `drop(Array("id","name"))`: Drops rows containing `null` or `NaN` value in **specified** columns, and returns a new DataFrame

### Replacing unwanted values with a constant

* `fill(0)`: Replaces all occurrences of `null` or `NaN` value in **numeric** columns with the specified value, and returns a new DataFrame.
* `fill(Map("minBalance" -> 0))`: Replaces all occurrences of `null` or `NaN` value in the **specified** column with the specified value, and returns a new DataFrame.
* `replace(Array("id"),Map(1234 -> 8923))`: Replaces **specified value** (1234) in the **specified column** (id) with the **specified value** (8923), and returns a new DataFrame.

## Common Actions on DataFrame


