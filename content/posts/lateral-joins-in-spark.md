---
title: "Lateral joins in Spark"
date: 2025-10-20T00:00:00
draft: false
categories: [Scala 2, Spark]
tags: [lateral join]
---

In this short article, I will explain lateral joins in Spark. 
I will demonstrate lateral joins using a simple example in Scala, and also show alternative approaches like inner join calculation with window functions and pure Spark SQL.

The `def lateralJoin(right: Dataset[_]): DataFrame` function on Dataset was introduced in version 4.0.0, but according to the documentation, the lateral subquery feature was first introduced in version [3.2.0](https://issues.apache.org/jira/browse/SPARK-34382).

Source code is located here: [https://github.com/JurajBurian/spark-playground](https://github.com/JurajBurian/spark-playground).

## What is a lateral join?

A lateral join is a type of join that performs a join operation between a table and a subquery. The subquery is executed for each row of the outer table, and the results are used to perform the join operation.

One can imagine that for each row from the outer table, the subquery may join several rows from the inner table.

This approach can be an alternative to window functions in many cases and may also provide better performance in certain scenarios.

## Model

Let's imagine that we have two datasets (tables) defined as follows:

```Scala
 def getDepartmentDF(implicit spark: SparkSession): DataFrame = {
  import spark.implicits._
  Seq(
    (1, "Engineering", 100000),
    (2, "Sales", 80000),
    (3, "Marketing", 90000)
  ).toDF("id", "department", "budget")
}

def getEmployeeDF(implicit spark: SparkSession): DataFrame = {
  import spark.implicits._
  Seq(
    (1, "John", 75000),
    (1, "Jane", 80000),
    (2, "Mike", 60000),
    (2, "Sarah", 65000),
    (3, "Tom", 70000),
    (3, "Ann", 64000),
    (2, "Moni", 84000),
    (2, "George", 85000),
    (2, "Klaudia", 85000)
  ).toDF("dept_id", "name", "salary")
}
```
or in SQL:
```SQL
CREATE TABLE department(id INT,dept_name VARCHAR(50),budget INT);
CREATE TABLE employee(dept_id INT,name VARCHAR(50),department VARCHAR(50),salary INT);
```

### Task 
Let calculate employees having the biggest salary per department and exceed budget as inner join.
Here is result:


| id|budget|   name|department|salary|
|---|------|-------|----------|------|
|  2| 80000| George|     Sales| 85000|
|  2| 80000|Klaudia|     Sales| 85000|
|  2| 80000|   Moni|     Sales| 84000|


### inner join approach 

```Scala
val employees   = getEmployeeDF
val rankedEmployees = employees
  .withColumn("rank", row_number().over(Window.partitionBy("dept_id").orderBy($"salary".desc)))
  .where($"rank" <= 3)

val result = departments
  .join(
    rankedEmployees,
    (departments("id") === rankedEmployees("dept_id")) and (departments("budget") < rankedEmployees("salary"))
  )
  .select("id", "budget", "name", "department", "salary")

```
As we can see, the code is a bit verbose. We need to calculate the rank for each employee and then select only employees with rank 1. Then we perform a join between departments and employees.

#### Pros and Cons
* (+) Potentially better performance - selects a subset of employees
* (-) Code is a bit verbose, and it's sometimes not easy to express complex logic through window functions
 
### Lateral join approach using Dataset

As we mentioned, the `lateralJoin` function is new in Spark 4.0.0.  

```Scala
val departments = getDepartmentDF.alias("ds")
val employees   = getEmployeeDF

val result = departments
.lateralJoin(
  employees
    .where(col("ds.id").outer === $"dept_id"))
    .orderBy($"salary".asc)
    .limit(1)
)
.select("id", "budget", "name", "department", "salary")
result.show()
```
#### Pros and Cons
* (+) Code is more concise. You can see the entire logic in one place: `orderBy` and `limit`, together with the join condition.
* (-) Performance may be worse because we need to perform a join for each row, but in many cases, proper repartitioning and caching can improve performance.

> __Note:__ Let's imagine that we have large datasets. A good strategy to improve performance is to repartition both datasets so that the final join will have the right subset of data close to each other. For example:
> ```scala
> val departments = getDepartmentDF.alias("ds").repartition(x, $"id")
> val employees   = getEmployeeDF.alias("es").repartition(x, $"dept_id")
>```
> where `x` is a reasonable number of partitions.

> __Note:__ As we can see, the departments table must be aliased, and in the WHERE condition, the alias must be used together with the `outer` keyword: `col("ds.id").outer`.

### Pure Spark SQL approach 

```Scala
getDepartmentDF.createTempView("ds")
getEmployeeDF.createTempView("es")

val result = spark
.sql(
  """
  |SELECT ds.id, ds.budget, es.name, ds.department, es.salary
  |FROM ds
  |INNER JOIN LATERAL (
  |  SELECT *
  |  FROM es
  |  WHERE ds.id = es.dept_id
  |  ORDER BY salary ASC
  |  LIMIT 1
  |) AS es
  |""".stripMargin
)
result.show()
```
or alternatively for INNER join:

```SQL
SELECT ds.id, ds.budget, es.name, ds.department, es.salary
FROM ds, LATERAL (
  SELECT *
  FROM es
  WHERE ds.id = es.dept_id
  ORDER BY salary ASC
  LIMIT 1
) AS es
```

The pure Spark SQL approach has the most concise form.

> __Remark__: INNER, CROSS, LEFT lateral join are supported in Spark SQL.

### When to Choose Lateral Joins in Spark

1. The subquery's logic depends on a value from each row of the main table.
2. Working with complex correlated subqueries.
3. Expanding JSON/array (or nested structures) into separate rows.

-----
> __Optimization:__ Spark's Catalyst optimizer applies logical optimizations to LATERAL joins (also known as correlated joins), such as predicate pushdown or join reordering. However, for efficient execution, ensure that:
> - Correlated predicates (e.g., `WHERE ds.id = es.dept_id`) are highly selective to minimize the number of rows processed per correlation.
> - The right-side table is efficiently scannable, ideally through partitioning, bucketing, or sorting on the join key to minimize data shuffling and enable broadcast joins if applicable.
> - Data distribution is optimized; repartitioning both tables on the join key can significantly improve performance by reducing shuffle overhead and data skew.