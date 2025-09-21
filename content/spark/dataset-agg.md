---
title: "3. Aggregation functions on Datasets"
draft: false
---

On Datasets one may call several aggregation functions directly.Those functions are shorcuts on top of [groupBy](/spark/rdd#groupBy) function. In next section we will see how those shortcut funcions are implemented.

#### agg shortcuts {#aggShortcuts}

```Scala
agg(aggExpr: (String, String), aggExprs: (String, String)*): DataFrame = {
    groupBy().agg(aggExpr, aggExprs: _*)}
```
```Scala
agg(exprs: Map[String, String]): DataFrame = groupBy().agg(exprs)
```
```Scala
agg(expr: Column, exprs: Column*): DataFrame = groupBy().agg(expr, exprs: _*)
```

example: 

```Scala
ds.groupBy().agg("age" -> "max", "salary" -> "avg")
ds.agg(Map("age" -> "max", "salary" -> "avg"))
import org.apache.spark.sql.functions.{avg, max} 
ds.agg(max($"age"), avg($"salary"))
```

