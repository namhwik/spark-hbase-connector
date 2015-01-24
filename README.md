# Spark-HBase Connector

This library lets your Apache Spark application interact with Apache HBase using a simple and elegant API.

If you want to read and write data to HBase, you don't need using the Hadoop API anymore, you can just use Spark.

## Including the library

The spark-hbase-connector library is not yet available in Maven repository, so you need to build it from the source code.

Check out this Github repo and execute the following command from the root folder:

    sbt package

SBT will create the library jar under `target/scala-2.10`.

Check also if the current branch is passing all tests in Travis-CI before checking out (See the following icon).

[![Build status](https://travis-ci.org/nerdammer/spark-hbase-connector.svg)](https://travis-ci.org/nerdammer/spark-hbase-connector)

## Writing to HBase (Basic)

Writing to HBase is very easy. Remember to import the implicit conversions:

```scala
import it.nerdammer.spark.hbase._
```

You have just to create a sample RDD, as the following one:

```scala
val rdd = sc.parallelize(1 to 100)
            .map(i => (i.toString, i+1, "Hello"))
```

This *rdd* is made of tuples like `("1", 2, "Hello")` or `("27", 28, "Hello")`. The first element of each tuple is considered the **row id**,
the others will be assigned to columns.

```scala
rdd.toHBaseTable("mytable")
    .toColumns("column1", "column2")
    .inColumnFamily("mycf")
    .save()
```

You are done. HBase now contains *100* rows in table *mytable*, each row containing two values for columns *mycf:column1* and *mycf:column2*.


## Reading from HBase (Basic)

Reading from HBase is easier. Remember to import the implicit conversions:

```scala
import it.nerdammer.spark.hbase._
```

Supposing you want to read the sample data you have written in previous example, you just need to write:

```scala
val hBaseRDD = sc.hbaseTable[(String, Int, String)]("mytable")
    .select("column1", "column2")
    .inColumnFamily("mycf")
```

Now *hBaseRDD* contains all data found in the table. Each object in the RDD is a tuple conaining (in order) the *row id*,
the corresponding value of *column1* (Int) and *column2* (String).

If you don't want the *row id* and want only to see the columns, just remove the first element from the tuple specs:

```scala
val hBaseRDD = sc.hbaseTable[(Int, String)]("mytable")
    .select("column1", "column2")
    .inColumnFamily("mycf")
```

This way, only the columns that you have chosen will be returned.

## Other Topics

### Filtering
It is possible to filter the results by prefixes of the row key. Filtering also supports additional salting prefixes
(see the [salting](#salting) section).

```scala
val rdd = sc.hbaseTable[(String, String)]("table")
      .select("col")
      .inColumnFamily(columnFamily)
      .withStartRow("00000")
      .withStopRow("00500")
```

The example above retrieves all rows having a row key *greater or equal* to `00000` and *lower* than `00500`.
The options `withStartRow` and `withStopRow` can also be used separately.

### Managing Empty Columns
Empty columns are managed by using `Option[T]` types:

```scala
val rdd = sc.hbaseTable[(Option[String], String)]("table")
      .select("column1", "column2")
      .inColumnFamily(columnFamily)

rdd.foreach(t => {
    if(t._1.nonEmpty) println(t._1.get)
})
```

You can use the `Option[T]` type every time you are not sure whether a given column is present in your HBase RDD.

### Using different column families
Different column families can be used both when reading or writing an RDD.

```scala
data.toHBaseTable("mytable")
      .toColumns("column1", "cf2:column2")
      .inColumnFamily("cf1")
      .save()
```

In the example above, `cf1` refers only to `column1`, because `cf2:column2` is already *fully qualified*.

```scala
val count = sc.hbaseTable[(String, String)]("mytable")
      .select("cf1:column1", "column2")
      inColumnFamily("cf2")
      .count
```

In the *reading example* above, the default column family `cf2` applies only to `column2`.

### Setting the HBase host
The HBase Zookeeper quorum host can be set in multiple ways.

(1) Passing the host to the `spark-submit` command:


    spark-submit --conf spark.hbase.host=thehost ...


(2) If you have access to the JVM parameters:


    java -Dspark.hbase.host=thehost -jar ....


(3) Using the *scala* code:


```scala
val sparkConf = new SparkConf()
...
sparkConf.set("spark.hbase.host", "thehost")
...
val sc = new SparkContext(sparkConf)
```

## Advanced

### Salting Prefixes<a name="salting"></a>

Salting is supported in reads and writes. Only *string valued row id* are supported at the moment, so salting prefixes
should also be of *String* type.

```scala
sc.parallelize(1 to 1000)
      .map(i => (pad(i.toString, 5), "A value"))
      .toHBaseTable(table)
      .inColumnFamily(columnFamily)
      .toColumns("col")
      .withSalting((0 to 9).map(s => s.toString))
      .save()
```

In the example above, each row id is composed of *5* digits: from `00001` to `01000`.
The *salting* property adds a random digit in front, so you will have records like: `800001`, `600031`, ...

When reading the RDD, you have just to declare the salting type used in the table and
ignore it when using bounds (startRow or stopRow). The library takes care of dealing with salting.

```scala
val rdd = sc.hbaseTable[String](table)
      .select("col")
      .inColumnFamily(columnFamily)
      .withStartRow("00501")
      .withSalting((0 to 9).map(s => s.toString))
```

### Custom Mapping
To be implemented

