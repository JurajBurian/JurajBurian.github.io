---
title: "2. Resilient Distributed Datasets"
draft: false
---

fault-tolerant collection of elements that can be operated on in parallel. We recognize `Dataset` and DataFrame. `DataFrame`,`DataFrame` is alias of `Dataset[Row]`.`Row` is equivalent of row in databases. 

```Scala
val rdd:Dataset[Int] = sc.parallelize(1 to 10)
val df:DataFrame = rdd.toDF()
```
### Conversions on Dataset
 
* __`as[U:Encoder]:Dataset[U]`__
* __`toDF():DataFrame`__
* __`rdd: RDD[T]`__

alternatives:

__`toDF(colNames:String*):DataFrame`__

__`to(schema:StructType):DataFrame`__
  

### Misc functions 

* __`col(colName:String):Column`__
  <br>alias:

  __`apply(colName:String):Column`__
  <br>alternatives:

  __`colRegex(colName:String):Column`__


* __`metadataColumn(colName:String):Column`__

<!-- TODO explanation -->

* __`as(alias:String):Dataset[T]`__ 
  <br>return named Dataset with an `alias`
  <br>alternatives:

  __`as(alias:Symbol):Dataset[T]`__

  __`alias(alias:String):Dataset[T]`__

  __`alias(alias:Symbol):Dataset[T]`__

* __`hint(name:String,parameters:Any*):Dataset[T]`__
  ```scala
  rdd.hint("broadcast","true")
  rdd.hint("rebalance",10)
  ```

* __`scalar():Column`__ 
  <br>return a Column object for a SCALAR Subquery containing exactly one row and one column.

* __`exists():Column`__
<br>return a Column object for an EXISTS Subquery.

* __`storageLevel:StorageLevel`__
<br>return the Dataset's current storage level, or `StorageLevel.NONE` if not persisted.

* __`createTempView(viewName: String): Unit`__
  <br>Creates a local temporary view using the given name. The lifetime of this temporary view is tied to the SparkSession that was used to create this Dataset.
  <br>variants:<br>
  __`createOrReplaceTempView(viewName: String): Unit`__

* __`createGlobalTempView(viewName: String): Unit`__
  <br>Creates a global temporary view using the given name. The lifetime of this temporary view is tied to this Spark application.
  <br>variants:<br>
  __`createOrReplaceGlobalTempView(viewName: String): Unit`__

* __`mergeInto(table: String, condition: Column): MergeIntoWriter[T]`__

* __`writeStream: DataStreamWriter[T]`__


### Tranformations

* __`join(right:Dataset[_]):DataFrame`__
  * behave as inner join 
  * need subsequent join predicate:
```scala
df1.join(df2).where(df1.col("id") === df2.col("id"))
```
alternatives:

  __`join(right:Dataset[_],usingColumn:String):DataFrame`__

  __`join(right:Dataset[_],usingColumns:Seq[String]):DataFrame`__

  __`join(right:Dataset[_],usingColumns:Array[String]):DataFrame`__

  __`join(right:Dataset[_],joinExprs:Column):DataFrame`__

  ```scala
    df1.join(df2,$"id" === $"id")
  ```
  __`crossJoin(right:Dataset[_]):DataFrame`__
  <br>Explicit cartesian join with another DataFrame. Don't use it if not necessary,operation is extremly expensive.
  <br>analogies with specification of join type (`inner`,`cross`,`outer`,`full`,`fullouter`,`full_outer`,`left`,`leftouter`,`left_outer`,`right`,`rightouter`,`right_outer`,`semi`,`leftsemi`,`left_semi`,`anti`,`leftanti`,`left_anti`)
  >for example:<br>
  >__`join(right:Dataset[_],usingColumns:Seq[String],joinType:String):DataFrame`__

* __`joinWith[U](other:Dataset[U],condition:Column,joinType:String):Dataset[(T,U)]`__
  <br>join that return tuple instead of dataframe,this is more native scala approach. 
  > supported join types :`inner`,`cross`,`outer`,`full`,`fullouter`,`full_outer`,`left`,`leftouter`,`left_outer`,`right`,`rightouter`,`right_outer`

* __`lateralJoin(right:Dataset[_]):DataFrame`__
  <br>alternatives:<br>
  __`lateralJoin(right:Dataset[_],joinExprs:Column):DataFrame`__<br>
  __`lateralJoin(right:Dataset[_],joinType:String):DataFrame`__<br>
  __`lateralJoin(right:Dataset[_],joinExprs:Column,joinType:String):DataFrame`__<br>

  > Lateral joins were implemented in version 4.0.0

 <!-- // TODO examples -->

* __`sortWithinPartitions(sortExprs:Column*):Dataset[T]`__
  ```scala
  rdd.sortWithinPartitions($"amount".desc)
  ```
  alternatives:<br>
  __`sortWithinPartitions(sortCol:String,sortCols:String*):Dataset[T]`__<br>
  __`sort(sortExprs:Column*):Dataset[T]`__<br>
  __`sort(sortCol:String,sortCols:String*):Dataset[T]`__<br>
  <br>aliases:<br>
  __`orderBy(sortCol:String,sortCols:String*):Dataset[T]`__<br>
  __`orderBy(sortExprs:Column*):Dataset[T]`__<br>

* __`select(cols:Column*):DataFrame`__
  ```scala
  rdd.select($"id",$"amount".as("total_amount"))
  ```
  alternatives:<br>
  __`selectExpr(exprs:String*):DataFrame`__
  ```scala
  rdd.selectExpr("id as id","amount as total_amount")
  ```
  __`select(col:String,cols:String*):DataFrame`__<br>
  __`select[U1](c1:TypedColumn[T,U1]):Dataset[U1]`__  
  
  > cardinality till 5 columns,here is example of declaration for 2 columns:
  >
  > __`select[U1,U2](c1:TypedColumn[T,U1],c2:TypedColumn[T,U2]):Dataset[(U1,U2)]`__

* __`filter(condition:Column):Dataset[T]`__
  ```scala
  rdd.filter($"count" >= 42)
  ```
  alternatives:<br>
  __`filter(conditionExpr:String):Dataset[T]`__
  ```scala
  rdd.filter("count >= 42")
  ```
  __`where(conditionExpr:String):Dataset[T]`__ - alias of `filter`

##### Transformations used before aggreggations returns RelationalGroupedDataset {#groupBy}

> `RelationalGroupedDataset` has own set of transformation methods 
<!-- TODO explanation of RelationalGroupedDataset methods -->


* __`groupBy(cols:Column*):RelationalGroupedDataset`__ 
  <br>alternatives:<br>
  __`groupBy(cols:Column*):RelationalGroupedDataset`__<br>
  __`groupBy(col:String,cols:String*):RelationalGroupedDataset`__<br>
  > see [agg shortcuts](/spark/rdd-agg#aggShortcuts)

* __`rollup(cols:Column*):RelationalGroupedDataset`__
  <br>alternatives:<br>
  __`rollup(col1:String,cols:String*):RelationalGroupedDataset`__

<!-- TODO examples -->  
* __`cube(cols:Column*):RelationalGroupedDataset`__
  <br>alternatives:<br>
  __`cube(col1:String,cols:String*):RelationalGroupedDataset`__

<!-- TODO examples -->
* __`groupingSets(groupingSets:Seq[Seq[Column]],cols:Column*):RelationalGroupedDataset`__


<!-- TODO move this section ^^^ else where as different chapter -->  
 
__`reduce(func:(T,T) => T):T`__  analogy to `reduce` in the collection API.


<!-- TODO examples -->
* __`groupByKey[K:Encoder](func:T=>K):KeyValueGroupedDataset[K,T]`__

<!-- TODO examples -->
* __`unpivot(ids:Array[Column],values:Array[Column],variableColumnName:String,valueColumnName:String):DataFrame`__
  <br>alternatives:<br>
  __`unpivot(ids:Array[Column],variableColumnName:String,valueColumnName:String):DataFrame`__
  > `melt` is an alias of `unpivot`

* __`transpose(indexColumn:Column):DataFrame`__
  ```Scala
  val df=Seq(("A",1,2),("B",3,4),("C",5,6")).toDF("id","x","y")
  df.show() 
  +---+---+---+
  | id|x  |y  |
  +---+---+---+
  |A  |1  |2  |
  |B  |3  |4  |
  |C  |5  |6  |
  +---+---+---+
  df.transpose("id").show()
  +---+---+---+---+
  |key| A | B | C |
  +---+---+---+---+
  | x | 1 | 3 | 5 |
  | y | 2 | 4 | 6 |
  +---+---+---+---+
  ```
  alternative:<br>
  __`transpose():DataFrame`__

* __`observe(name:String,expr:Column,exprs:Column*):Dataset[T]`__
  <br>alternative:<br>
  __`observe(observation:Observation,expr:Column,exprs:Column*):Dataset[T]`__
<!-- TODO example -->


* __`limit(n:Int):Dataset[T]`__ 
  <br>return limited `Dataset` to first n rows

* __`offset(n:Int):Dataset[T]`__
  <br>return `Dataset` without first n rows 

* __`union(other:Dataset[T]):Dataset[T]`__ 
    > `unionAll` is an alias of `union`
  
    >This function resolves columns by their positions in the schema, not the fields in the strongly typed objects.

* __`unionByName(other:Dataset[T],allowMissingColumns:Boolean):Dataset[T]`__
  <br>alternative:<br> 
  __`unionByName(other:Dataset[T]):Dataset[T]`__ - allows missing columns = false

* __`intersect(other:Dataset[T]):Dataset[T]`__
  <br> INTERSECT equivalnet in SQL 

* __`intersectAll(other:Dataset[T]):Dataset[T]`__
  <br>INTERSECT ALL equivalnet in SQL 

* __`except(other:Dataset[T]):Dataset[T]`__
  <br>EXCEPT equivalnet in SQL 

* __`exceptAll(other:Dataset[T]):Dataset[T]`__
  <br>EXCEPT ALL equivalnet in SQL 


* __`sample(withReplacement:Boolean,fraction:Double,seed:Long):Dataset[T]`__
  <br>alternative:<br>
  __`sample(withReplacement:Boolean,fraction:Double):Dataset[T]`__

* __`randomSplit(weights:Array[Double],seed:Long):Array[Dataset[T]]`__
  <br>alternative:<br>
  __`randomSplit(weights:Array[Double]):Array[Dataset[T]]`__
  <br>variant:<br>
  __`randomSplitAsList(weights:Array[Double],seed:Long):util.List[Dataset[T]]`__

* __`withColumn(colName:String,col:Column):DataFrame`__
  <br>alternative:<br>
  __`withColumns(colsMap:Map[String, Column]):DataFrame`__

* __` withColumnRenamed(existingName:String,newName:String):DataFrame`__
  <br>alternative:<br>
  __`withColumnsRenamed(colsMap:Map[String,String]):DataFrame`__

* __`withMetadata(columnName:String,metadata:Metadata):DataFrame`__

* __`drop(colNames:String*):DataFrame`__
  <br>drop columns
  <br>alternative:<br>
  __`drop(col:Column,cols:Column*):DataFrame`__
  <br>variant:<br>
  __`drop(colName:String):DataFrame`__

* __`dropDuplicates():Dataset[T]`__
  <br>returns a new Dataset that contains only the unique rows from this Dataset. This is an alias for `distinct`.
  <br>alternatives:<br>
  __`dropDuplicates(colNames:Seq[String]):Dataset[T]`__
  __`dropDuplicates(colNames:Array[String]):Dataset[T]`__<br>
  __`dropDuplicates(col1:String,cols:String*):Dataset[T]`__
  <br>synonym:<br>
  __`distinct():Dataset[T]`__

* __`dropDuplicatesWithinWatermark():Dataset[T]`__
  <br>Returns a new Dataset with duplicates rows removed, within watermark. This only works with streaming Dataset, and watermark for the input Dataset must be set via withWatermark.
  <br>alternatives:<br>
  __`dropDuplicatesWithinWatermark(colNames:Seq[String]):Dataset[T]`__
  __`dropDuplicatesWithinWatermark(colNames:Array[String]):Dataset[T]`__<br>
  __`dropDuplicatesWithinWatermark(col1:String,cols:String*):Dataset[T]`__

* __`describe(cols:String*):DataFrame`__
  <br>computes basic statistics for numeric and string columns, including count, mean, stddev, min, and max. If no columns are given, this function computes statistics for all numerical or string columns.

* __`summary(statistics:String*):DataFrame`__

* __`head:(n:Int):Array[T]`__
  <br>Returns the first n rows.
  <br>variants:<br>
  __`head:():T`__<br>
  __`first():T`__
  <br>synonym:<br>
  __` take(n:Int):Array[T]`__

* __`transform[U,DSO[_]<:Dataset[_]](t:this.type=>DSO[U]):DSO[U]`__

* __`map[U:Encoder](func:T=>U):Dataset[U]`__
  <br>alternatives:<br>
  __`mapPartitions[U:Encoder](func:Iterator[T] => Iterator[U]):Dataset[U]`__

* __`flatMap[U:Encoder](func:T => IterableOnce[U]):Dataset[U]`__

* __`foreach(f:T => Unit):Unit`__
  <br>alternatives:<br>
  __`foreachPartition(f:Iterator[T] => Unit):Unit`__

* __`tail(n:Int):Array[T]`__

* __`collect():Array[T]`__

<!-- TODDO exists scala version ? -->
* __`toLocalIterator():util.Iterator[T]`__  

* __`count():Long`__

* __`repartition(numPartitions:Int):Dataset[T]`__
  <br>returns a new Dataset that has exactly numPartitions partitions.
  <br>variants:<br>
  __`def repartitionByExpression(numPartitions:Option[Int],partitionExprs:Seq[Column]):Dataset[T]`__
  __`repartition(numPartitions:Int, partitionExprs:Column*):Dataset[T]`__
  __`repartition(partitionExprs:Column*):Dataset[T]`__

* __`repartitionByRange(numPartitions: Int, partitionExprs: Column*): Dataset[T]`__
  <br>variants:<br>
  __`repartitionByRange(partitionExprs: Column*): Dataset[T]`__

* __`coalesce(numPartitions: Int): Dataset[T]`__

* __`persist(newLevel: StorageLevel): Dataset[T]`__
  > newLevel can be `MEORY_ONLY`, `MEMORY_AND_DISK`, `MEMORY_ONLY_SER`, `MEMORY_AND_DISK_SER`, `DISK_ONLY`, `MEMORY_ONLY_2`, `MEMORY_AND_DISK_2`, etc. See [StorageLevel](https://spark.apache.org/docs/latest/api/scala/org/apache/spark/storage/StorageLevel.html) documentation.
  <br>variants:<br>
  __`persist():Dataset[T]`__, __`cache(): Dataset[T]`__ - used `MEMORY_AND_DISK`

* __`unpersist(blocking: Boolean): Dataset[T]`__
  <br>mark the Dataset as non-persistent, and remove all blocks for it from memory and disk. This will not un-persist any cached data that is built upon this Dataset.
  <br>variants:<br>
  __`unpersist(): Dataset[T]`__


* __`toJSON: Dataset[String]`__

* __``__

* __``__



