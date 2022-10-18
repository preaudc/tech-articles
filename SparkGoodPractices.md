# Spark - good practices: some common caveats and solutions

## never collect a Dataset

## evaluate as little as possible

## avoid unnecessry SparkSession parameter

## remove extra columns when mapping a Dataset to a case class with fewer columns

## always specify schema when reading file (parquet, json or csv) into a DataFrame

## avoid union performance penalties when reading parquet files

## prefer select over withColumn when adding multiple columns

## use built-in functions rather than UDF

## manage wisely the number of partitions

## deactivate unnecessary cache

## (Scala) Prefer immutable variables
