# Null safe equi-join

The function [join](https://spark.apache.org/docs/latest/api/scala/org/apache/spark/sql/Dataset.html#join(right:org.apache.spark.sql.Dataset[_],usingColumns:Seq[String],joinType:String):org.apache.spark.sql.DataFrame) is an equi-join, meaning the join of two dataframes is made on a sequence of common columns of these two dataframes.
In other words, doing an equi-join is only possible if the join columns of the two dataframes **have the exact same name**.

Let's illustrate this with an example:

```scala
val df1 = Seq(
  ("1L", "aaa", 111),
  ("2L", "bbb", 222),
  ("3L", "ccc", 333),
  ("4L", "ddd", 444),
  ("5L", "eee", 555),
  ("6L", "fff", 666),
  ("7L", null, 777),
  ("8L", "hhh", 888)
).toDF("id1", "col_a", "col_b")
df1.show
+---+-----+-----+
|id1|col_a|col_b|
+---+-----+-----+
| 1L|  aaa|  111|
| 2L|  bbb|  222|
| 3L|  ccc|  333|
| 4L|  ddd|  444|
| 5L|  eee|  555|
| 6L|  fff|  666|
| 7L| NULL|  777|
| 8L|  hhh|  888|
+---+-----+-----+

val df2 = Seq(
  ("11L", "aaa", 111),
  ("33L", "ccc", 333),
  ("55L", "eee", 555),
  ("77L", null, 777)
).toDF("id2", "col_a", "col_b")
df2.show
+---+-----+-----+
|id2|col_a|col_b|
+---+-----+-----+
|11L|  aaa|  111|
|33L|  ccc|  333|
|55L|  eee|  555|
|77L| NULL|  777|
+---+-----+-----+

// Equi-join between df1 and df2 using a sequence of columns.
df1.join(df2, Seq("col_a", "col_b"), "inner").show
+-----+-----+---+---+
|col_a|col_b|id1|id2|
+-----+-----+---+---+
|  aaa|  111| 1L|11L|
|  ccc|  333| 3L|33L|
|  eee|  555| 5L|55L|
+-----+-----+---+---+
```

This method has several advantages over the same join expressed using a join expression:
```scala
df1.join(df2, df1("col_a") === df2("col_a") && df1("col_b") === df2("col_b"), "inner").show
+---+-----+-----+---+-----+-----+
|id1|col_a|col_b|id2|col_a|col_b|
+---+-----+-----+---+-----+-----+
| 1L|  aaa|  111|11L|  aaa|  111|
| 3L|  ccc|  333|33L|  ccc|  333|
| 5L|  eee|  555|55L|  eee|  555|
+---+-----+-----+---+-----+-----+
```
- The syntax is clearer and more straightforward.
- The join columns will only appear once in the output.

However, the equality test is not [null safe](https://spark.apache.org/docs/latest/api/scala/org/apache/spark/sql/Column.html#%3C=%3E(other:Any):org.apache.spark.sql.Column), meaning in our example that the row of df1 with `id1 == 7L` will not be joined to the row of df2 with `id2 == 77L` (because for the [standard equality test](https://spark.apache.org/docs/latest/api/scala/org/apache/spark/sql/Column.html#===(other:Any):org.apache.spark.sql.Column), `(null, 777) != (null, 777)`).

**&rarr; As a consequence, we will now try to implement a null safe equi-join.**

Let's first try with a join expression using null safe equality tests:
```scala
// Null safe equi-join
// Join between df1 and df2 using a join expression with null safe equality tests between columns.
df1.join(df2, df1("col_a") <=> df2("col_a") && df1("col_b") <=> df2("col_b"), "inner").show
+---+-----+-----+---+-----+-----+
|id1|col_a|col_b|id2|col_a|col_b|
+---+-----+-----+---+-----+-----+
| 1L|  aaa|  111|11L|  aaa|  111|
| 3L|  ccc|  333|33L|  ccc|  333|
| 5L|  eee|  555|55L|  eee|  555|
| 7L| NULL|  777|77L| NULL|  777|
+---+-----+-----+---+-----+-----+
```
We can see that:
- The equality test is null safe, i.e. `(null, 777) == (null, 777)`.
- But the join columns appear twice in the output.

Let's now try to implement it so that:
- It can be generalized to any sequence of columns.
- The join columns appear only once in the output.
```scala
import org.apache.spark.sql.{Column, DataFrame}
def joinNullSafe(leftDF: DataFrame, rightDF: DataFrame, usingColumns: Seq[String], joinType: String): DataFrame = {
  val joinExprs = forall(array(usingColumns.map(c => leftDF(c) <=> rightDF(c)):_*), identity)
  leftDF.join(rightDF, joinExprs, joinType)
    .select((usingColumns.map(c => leftDF(c)) ++ (leftDF.columns ++ rightDF.columns).filterNot(usingColumns.contains(_)).map(col)):_*)
}

joinNullSafe(df1, df2, Seq("col_a", "col_b"), "inner").show
+-----+-----+---+---+
|col_a|col_b|id1|id2|
+-----+-----+---+---+
|  aaa|  111| 1L|11L|
|  ccc|  333| 3L|33L|
|  eee|  555| 5L|55L|
| NULL|  777| 7L|77L|
+-----+-----+---+---+
```
**&rarr; We now effectively have implemented a null safe equi-join!**

## Appendix

Here is an alternative version of the null safe equi-join, which is based on the property that the equality test between two array of columns is null safe by default!
```scala
// Null safe equi-join - alternative solution
// Join between df1 and df2 using a join expression with an equality test between array of columns.
df1.join(df2, array(df1("col_a"), df1("col_b")) === array(df2("col_a"), df2("col_b")), "inner").show
+---+-----+-----+---+-----+-----+
|id1|col_a|col_b|id2|col_a|col_b|
+---+-----+-----+---+-----+-----+
| 1L|  aaa|  111|11L|  aaa|  111|
| 3L|  ccc|  333|33L|  ccc|  333|
| 5L|  eee|  555|55L|  eee|  555|
| 7L| NULL|  777|77L| NULL|  777|
+---+-----+-----+---+-----+-----+
```
We can see that:
- The equality test is null safe, i.e. `(null, 777) == (null, 777)`.
- But the join columns appear twice in the output.

This can also be implemented in a more generic way which also deduplicates join columns:
```scala
import org.apache.spark.sql.{Column, DataFrame}
def joinNullSafe(leftDF: DataFrame, rightDF: DataFrame, usingColumns: Seq[String], joinType: String): DataFrame = {
  val joinExprs = array(usingColumns.map(c => leftDF(c)):_*) === array(usingColumns.map(c => rightDF(c)):_*)
  leftDF.join(rightDF, joinExprs, joinType)
    .select((usingColumns.map(c => leftDF(c)) ++ (leftDF.columns ++ rightDF.columns).filterNot(usingColumns.contains(_)).map(col)):_*)
}
joinNullSafe(df1, df2, Seq("col_a", "col_b"), "inner").show
+-----+-----+---+---+
|col_a|col_b|id1|id2|
+-----+-----+---+---+
|  aaa|  111| 1L|11L|
|  ccc|  333| 3L|33L|
|  eee|  555| 5L|55L|
| NULL|  777| 7L|77L|
+-----+-----+---+---+
```
