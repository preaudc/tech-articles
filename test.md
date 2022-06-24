## avoid unnecessary SparkSession parameter

It is not necessary to pass the SparkSession as a function parameter if this function already has a Dataset[T] or DataFrame parameter.
Indeed, a Dataset[T] or DataFrame already contains a reference to the SparkSession.

For example:
```scala
def f(ds: Dataset[String], spark: SparkSession) = {
  import spark.implicits._
  // ...
}
```
can be replaced by;
```scala
def f(ds: Dataset[String]) = {
  import ds.sparkSession.implicits._
  // ...
}
```

Similarly:
```scala
def f(df: DataFrame, spark: SparkSession) = {
  import spark.implicits._
  // ...
}
```
can be replaced by;
```scala
def f(df: DataFrame) = {
  import df.sparkSession.implicits._
  // ...
}
```

## prefer select over withColumn when adding multiple columns

`withColumn` should be avoided when adding multiple columns. Quoted from [Spark source code](https://github.com/apache/spark/blob/v3.3.0/sql/core/src/main/scala/org/apache/spark/sql/Dataset.scala#L2472-L2475):

```
[`withColumn`] introduces a projection internally. Therefore, calling it multiple times,
for instance, via loops in order to add multiple columns can generate big plans which
can cause performance issues and even `StackOverflowException`. To avoid this,
use `select` with the multiple columns at once.`
```

Let's illustrate this by adding columns to `df`:
```scala
val df = Seq((1L, "a"), (2L, "b"), (3L, "c")).toDF("uid", "desc")
df.show
// will output:
// +---+----+
// |uid|desc|
// +---+----+
// |  1|   a|
// |  2|   b|
// |  3|   c|
// +---+----+
```

Add two columns "col1" and "col2" with `withColumn`:
```scala
df.withColumn("col1", lit("val1"))
  .withColumn("col2", lit("val2"))
  .show
// will output:
// +---+----+----+----+
// |uid|desc|col1|col2|
// +---+----+----+----+
// |  1|   a|val1|val2|
// |  2|   b|val1|val2|
// |  3|   c|val1|val2|
// +---+----+----+----+
```

Add two columns "col1" and "col2" with `select`:
```scala
df.select((
      df.columns.map(col(_)) :+   // get list of columns in df as a Seq[Column]
      lit("val1").as("col1") :+   // add Column "col1"
      lit("val2").as("col2")      // add Column "col2"
    ):_*                          // un-pack the Seq[Column] into arguments 
  ).show
// will output:
// +---+----+----+----+
// |uid|desc|col1|col2|
// +---+----+----+----+
// |  1|   a|val1|val2|
// |  2|   b|val1|val2|
// |  3|   c|val1|val2|
// +---+----+----+----+
```

N.B.: always prefer the `select` implementation when adding multiple columns!

## use built-in functions rather than UDF

## manage wisely the number of partitions

## deactivate unnecessary cache

## (Scala) Prefer immutable variables

## About the author

Christophe Préaud is Lead data engineer & technical referent in the data-platform team at Kelkoo Group.

You can connect with him on [LinkedIn](https://www.linkedin.com/in/christophe-pr%C3%A9aud-184023155).
