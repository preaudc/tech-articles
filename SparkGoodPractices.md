# Spark - good practices: some common caveats and solutions

##never collect a dataset
##evaluate as little as possible
From the Spark ScalaDoc
`Operations available on Datasets are divided into transformations and actions. Transformations are the ones that produce new Datasets, and actions are the ones that trigger computation and return results
(...)
Datasets are "lazy", i.e. computations are only triggered when an action is invoked`

In other words, you should limit the number of actions you apply on your Dataset, since each action will trigger a costly computation, while all transformations are lazily evaluated.

Though it is not alway possible, the ideal would be to call an action on your Dataset only once. Nevertheless, avoid calling an action on your Dataset unless it is necessary.

For example, doing a count (which is an action) for log printing should be avoided.

```scala
ds is computed twice in the exemple below
val ds = ds.map({ line =>
    line.copy(desc = s"this is a ${line.desc}")
})
println("num elements: " + ds.count)
ds.collect.foreach(println)
```
----------------------------------------------------------------------------------------------------
The simplest and preferable solution would be to skip the print, but if it is absolutely necessary you can use accumulators instead
`scala
val accum = sc.longAccumulator("num elements in ds")
val ds = ds.map({ line =>
    accum.add(1)
    line.copy(desc = s"this is a \${line.desc}")
})

ds.collect.foreach(println)
println("num elements: " + accum.value)
```

##use built-in functions rather than UDF
##manage wisely the number of partitions
##deactivate unnecessary cache
##always specify schema when reading file (parquet, json or csv) into a DataFrame
##avoid union performance penalties
##prefer select over withColumn
##(Scala/Java) remove extra columns when converting to a Dataset by providing a class
##(Scala) Prefer immutable variables
