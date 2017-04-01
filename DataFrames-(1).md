So far we saw what `DataFrame`s are, how to create them and how to use SQL queries on them.

**`DataFrame`s have their own APIs as well!**

## DataFrames Data Types

To enable optimization, Spark SQL's `DataFrame`s operate on a restricted set of data types.

```
Scala Type                 SQL Type            Details
---------------------------------------------------------------------------------------------------------
Byte                       ByteType            1 byte signed integers (-128, 127)
Short                      ShortType           2 byte signed integers (-32768, 32767)
Int                        IntegerType         4 byte signed integers (-2147483648, 2147483647)
Long                       LongType            8 byte signed integers
java.math.BigDecimal       DecimalType         Arbitrary precision signed decimals
Float                      FloatType           4 byte floating point number 
Double                     DoubleType          8 byte floating point number     
Array[Byte]                BinaryType          Byte sequence values        
Boolean                    BooleanType         true/false       
Boolean                    BooleanType         true/false       
java.sql.Timestamp         TimestampType       Date containing year, month, day, hour, minute, second
java.sql.Date              DateType            Date containing year, month, day          
String                     StringType          Character string values (stored as UTF8)     
```

Complex Spark SQL Data Types:
```
Scala Type                 SQL Type
-------------------------------------------------------------------
Array[T]                   ArrayType(elementType, containsNull)
Map[K, V]                  MapType(keyType, valueType, valueContainsNull)
case class                 StructType(List[StructFields])
```

**Arrays**

Array of only one type of element (`elementType`). 
`containsNull` is set to `true` if the elements in `ArrayType` value can have null values

E.g.
```scala
// scala type           // sql type
Array[String]            ArrayType(StringType, true)
```

**Maps**

Map of key/value pairs with two type of elements.
`valuecontainsNull` is set to `true` if the elements in `MapType` value can have null values.

E.g.
```scala
// scala type           // sql type
Map[Int, String]        Map(IntegerType, StringType, true)
```
**Structs**

Struct type with the list of possible fields of different types.
`containsNull` is set to `true` if the elements in `StructType` can have null values.

E.g.
```scala
// scala type                                     // sql type
case class Person(name: String, age: Int)         StructType(List(StructField("name", StringType, true)
                                                                  StructField("age", StringType, true)))
```