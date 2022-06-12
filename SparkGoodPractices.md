# Spark - good practices: some common caveats and solutions

## never collect a dataset

Never collect - quoted from the scaladoc:

`(collect) should only be used if the resulting array is expected to be small, as all the data is loaded into the driver's memory`

```scala
ds.collect()  // NOT OK - is long and has a good change of crashing the driver, unless ds is small

ds.show()     // OK - displays only the first 20 rows of ds

ds.write.parquet(...)  // OK - saves the content of ds out into external storage
```

## evaluate as little as possible

From the Spark ScalaDoc:

`Operations available on Datasets are divided into transformations and actions. Transformations are the ones that produce new Datasets, and actions are the ones that trigger computation and return results
(...)
Datasets are "lazy", i.e. computations are only triggered when an action is invoked`

In other words, you should limit the number of actions you apply on your Dataset, since each action will trigger a costly computation, while all transformations are lazily evaluated.

Though it is not alway possible, the ideal would be to call an action on your Dataset only once. Nevertheless, avoid calling an action on your Dataset unless it is necessary.

For example, doing a count (which is an action) for log printing should be avoided.

`ds` is computed twice in the exemple below:
```scala
case class Data(id: Long, desc: String, price: Double)
val ds = Seq(Data(1, "a", 123), Data(2, "b", 234), Data(3, "c", 345), Data(4, "a", 234), Data(5, "a", 345), Data(6, "b", 123), Data(7, "a", 234), Data(8, "c", 345), Data(9, "a", 234), Data(10, "b", 234)).toDS

val dsOut = ds.map({ line =>
   line.copy(desc = s"this is a '${line.desc}'")
})
println(s"num elements: ${dsOut.count}")  // first ds computation
dsOut.show                                // second ds computation
```
The simplest and preferable solution would be to skip the print, but if it is absolutely necessary you can use accumulators instead:
```scala
val accum = sc.longAccumulator("num elements in ds")
val dsOut = ds.map({ line =>
   accum.add(1)
   line.copy(desc = s"this is a '${line.desc}'")
})

dsOut.show  // single ds computation
println(s"num elements: ${accum.value}")
```

## use built-in functions rather than UDF

## manage wisely the number of partitions

## deactivate unnecessary cache

## always specify schema when reading file (parquet, json or csv) into a DataFrame

Let's begin with a Dataset mapped on case class `TestData`:
```scala
case class TestData(id: Long, desc: String)
import spark.implicits._
import org.apache.spark.sql.SaveMode
val ds = Seq(TestData(1L, "a"),TestData(2L, "b"),TestData(3L, "c"),TestData(4L, "d"),TestData(5L, "e")).toDS
ds.show
// will output:
// +---+----+
// | id|desc|
// +---+----+
// |  1|   a|
// |  2|   b|
// |  3|   c|
// |  4|   d|
// |  5|   e|
// +---+----+

// save ds locally
val localFile = "file:///home/cpreaud/output/test_schema_change"
ds.repartition(1).write.mode(SaveMode.Overwrite).parquet(localFile)
```
Now let's add a new field comment in case class `TestData`
```scala
case class TestData(id: Long, desc: String, comment: String)
```
We hit an error when we try to map the parquet file dataset to the new definition of `TestData`, because the schema of the parquet file and the schema of TestData do not match anymore:
```scala
val dsRead = spark.read.parquet(localFile).as[TestData]
// will output:
// org.apache.spark.sql.AnalysisException: cannot resolve 'comment' given input columns: [desc, id]
//  at org.apache.spark.sql.catalyst.analysis.package$AnalysisErrorAt.failAnalysis(package.scala:54)
// (...)
```
It works correctly if the schema is enforced when the parquet file is read:
```scala
import org.apache.spark.sql.Encoders
val schema = Encoders.product[TestData].schema
val dsRead = spark.read.schema(schema).parquet("file:///home/cpreaud/output/test_schema_change").as[TestData]
dsRead.show
// will output:
// +---+----+-------+
// | id|desc|comment|
// +---+----+-------+
// |  1|   a|   null|
// |  2|   b|   null|
// |  3|   c|   null|
// |  4|   d|   null|
// |  5|   e|   null|
// +---+----+-------+
```

## avoid union performance penalties

## prefer select over withColumn

## (Scala/Java) remove extra columns when converting to a Dataset by providing a class

## (Scala) Prefer immutable variables
