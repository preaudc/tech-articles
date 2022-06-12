# Spark - good practices: some common caveats and solutions

## never collect a dataset

Never collect - quoted from the scaladoc:

`(collect) should only be used if the resulting array is expected to be small, as all the data is loaded into the driver's memory`

```scala
ds.collect()  // NOT OK - will be long and have a good change of crashing the driver, unless ds is small

ds.show()     // OK - will only display the first 20 rows of ds

ds.write.parquet(...)  // OK - will save the content of ds out into external storage
```

## evaluate as little as possible

From the Spark ScalaDoc:

`Operations available on Datasets are divided into transformations and actions. Transformations are the ones that produce new Datasets, and actions are the ones that trigger computation and return results
(...)
Datasets are "lazy", i.e. computations are only triggered when an action is invoked`

In other words, you should limit the number of actions you apply on your Dataset, since each action will trigger a costly computation, while all transformations are lazily evaluated.

Though it is not alway possible, the ideal would be to call an action on your Dataset only once. Nevertheless, avoid calling an action on your Dataset unless it is necessary.

For example, doing a count (which is an action) for log printing should be avoided.

ds is computed twice in the exemple below
```scala
case class Data(id: Long, desc: String, price: Double)
val ds = Seq(Data(1, "a", 123), Data(2, "b", 234), Data(3, "c", 345), Data(4, "a", 234), Data(5, "a", 345), Data(6, "b", 123), Data(7, "a", 234), Data(8, "c", 345), Data(9, "a", 234), Data(10, "b", 234)).toDS

val dsOut = ds.map({ line =>
   line.copy(desc = s"this is a '${line.desc}'")
})
println(s"num elements: ${dsOut.count}")  // first ds computation
dsOut.show                                // second ds computation
```
----------------------------------------------------------------------------------------------------
The simplest and preferable solution would be to skip the print, but if it is absolutely necessary you can use accumulators instead
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

## avoid union performance penalties

## prefer select over withColumn

## (Scala/Java) remove extra columns when converting to a Dataset by providing a class

## (Scala) Prefer immutable variables
