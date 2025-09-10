---
title: "Developing Spark jobs in Scala"
date: 2025-08-11T00:00:00
draft: false
categories: [Scala 2, Spark]
tags: [sbt, munit, testcontainers, Kafka]
---

In this short article, I would like to cover the entire development cycle of Spark jobs. I will explain following topics:

1. setup of sbt multi-module project
2. how to write modular code
3. how to write testable code
4. how to write unit tests
5. how to write integration tests

Source code is located here: [https://github.com/JurajBurian/spark-example](https://github.com/JurajBurian/spark-example) .

## The sbt project. 

In this sample project, we have defined a single Spark job as a module alongside a common module to demonstrate modular development. This approach is useful when developing several Spark jobs, that share models or functionality.

_build.sbt_
```Scala
val v = new {
  val Scala          = "2.13.16"
  val Spark          = "4.0.0"
  val Munit          = "1.1.1"
  val Testcontainers = "1.21.3"
}

ThisBuild / version      := "0.1.0-SNAPSHOT"
ThisBuild / scalaVersion := v.Scala
ThisBuild / scalacOptions ++=
  Seq("-encoding", "UTF-8", "-unchecked", "-feature", "-explaintypes")

val coreSparkLibs = Seq(
  "org.apache.spark" %% "spark-core" % v.Spark % Provided,
  "org.apache.spark" %% "spark-sql"  % v.Spark % Provided
)
val commonLibs = Seq(
  "org.scalameta" %% "munit" % v.Munit % Test
)

lazy val `spark-examples-root` = (project in file("."))
  .settings(publishLocal := {}, publish := {}, publishArtifact := false)
  .aggregate(`common`, `example-1`)

lazy val `common` = (project in file("common")).settings(
  libraryDependencies ++= commonLibs
)

lazy val `example-1` = (project in file("example-1"))
  .settings(
    assembly / mainClass := Some("com.jubu.spark.Main"),
    libraryDependencies ++= coreSparkLibs ++ commonLibs ++ Seq(
      "org.testcontainers" % "kafka"                      % v.Testcontainers % Test,
      "org.apache.spark"  %% "spark-sql-kafka-0-10"       % v.Spark,
      "org.apache.spark"  %% "spark-streaming-kafka-0-10" % v.Spark
    ),
    // Assembly settings
    assembly / assemblyJarName := "spark-example-1.jar",
    assembly / assemblyOption ~= { _.withIncludeScala(false) },
    run / fork    := true,
    Test / fork   := true,
    Compile / run := Defaults
      .runTask(Test / fullClasspath, Compile / run / mainClass, Compile / run / runner)
      .evaluated
  )
  .dependsOn(`common`)
```

Our project follows a standard layout: `spark-example-root` serving as the parent module that aggregates both the common and the `example-1` modules. We divided definitions of dependencies to several groups. Important is that `coreSparkLibs` contains dependencies annotated as `Provided`. Provided dependency is not part of runtime classpath, but also is not a part of uber-jar (jar containing all dependencies necessary for running Spark job - we will talk about uber-jar later). 

> *Remark:* dependency annotated as Test is accessible only in test phase. Provided dependency is part of classpath in test phase! 

_project/plugins.sbt_
```Scala
addSbtPlugin("com.eed3si9n"  % "sbt-assembly" % "2.3.1")
addSbtPlugin("org.scalameta" % "sbt-scalafmt" % "2.5.2")
```

In plugins section is mentioned `sbt-assembly` plugin responsible for building uber jar. Call of sbt assembly triggers build of uber-jar that is used in production. The configuration of uber-jar is on lines 38 and 39 (in build.sbt) - we exclude Scala libraries from uber-jar.

Let talk about forking. On lines 40 and 41 (in build.sbt) we define the `fork` for `Test` and `run` scope. It is **crucial** in cases when we want to execute Spark job during tests or just local run. The reason for that is that more complex jobs do not clean resources correctly, namely in our example some Kaka threads keep working after test is finished. For simple unit testing is a fork redundant - just causes slover testing (jvm invocation time) - see common module.

>Lines 42-44 (in build.sbt) are important. We want run the job locally, then we need classpath that includes also provided dependencies. Overriding run task classpath with classpath from test scope is a way, how to put also provided dependencies to run task.

## Modular code

When is it time to divide a project into modules? There are several reasons:

* A module can define an **API, common code**, or a **subdomain** of the overall domain.
* A modular build structure significantly reduces project complexity.
* It establishes the foundation for future service extraction if the project grows substantially. Maybe this argument is more valid in application or service oriented projects.

In the common module, we defined only one function to demonstrate the purpose:

```Scala
package jubu

package object spark {
  /**
   * Splits a string into a sequence of lowercased words.
   */
  val splitter: String => Seq[String]  = { p:String =>
    p.split("\\s+").map(_.toLowerCase).toSeq
  }
}
```

## Testable code

It is always recommended to write unit tests when functionality becomes complex or when interactions between code blocks reach a certain level of complexity. On the other hand, integration tests validate interactions between multiple systems and often verify the behavior of the entire system. I prefer integration tests over mocked tests, as they test real system interactions. Our hardware is powerful enough to handle even complex systems with multiple integrations. I use `munit` for testing.

Here we test splitter function as demonstration of unit testing.

But how do we write Spark tasks that have in balance simplicity and testability? How can we create code that's verifiable both in isolation and as an integrated whole? Let's explore this step by step.

`jubu/spark/Calculation.scala`
```Scala 3
package jubu.spark

import org.apache.spark.sql.streaming.DataStreamWriter
import org.apache.spark.sql.{DataFrame, Dataset, Row, SparkSession}

object Calculation {

  def source(spark: SparkSession, brokers: String, topics: String): Dataset[String] = {
    import spark.implicits._
    val df = spark.readStream
      .format("kafka")
      .option("kafka.bootstrap.servers", brokers)
      .option("subscribe", topics)
      .option("startingOffsets", "earliest")
      .load()
    df.selectExpr("CAST(value AS STRING)").as[String]
  }

  def aggregation(spark: SparkSession, dataset: Dataset[String]): DataFrame = {
    import spark.implicits._
    dataset.flatMap(splitter).groupBy("value").count()
  }

  def sink(dataset: DataFrame, brokers: String): DataStreamWriter[Row] = {
    // dataset.writeStream.outputMode("complete").format("console")
    dataset
      .selectExpr("CAST(value AS STRING) AS key", "CAST(count AS STRING) AS value")
      .writeStream
      .format("kafka")
      .option("kafka.bootstrap.servers", brokers)
      .option("topic", "output-topic")
      .option("checkpointLocation", "/tmp/checkpoint")
      .outputMode("complete")
  }

  def build(spark: SparkSession, brokers: String, topics: String): DataStreamWriter[Row] = {
    val src    = source(spark, brokers, topics)
    val result = aggregation(spark, src)
    sink(result, brokers)
  }
}
```

#### Recommended Structure for Testable Spark Code

I recommend structuring the code to clearly separate:

1. The **source** (data input)
2. The **algorithm** (aggregation logic)
3. The **consumer** (result output)

These components should then be combined in a _build_ method. Of course, real-world scenarios are often more complex and require careful consideration of which parts of the computation require specialized testing and should be isolated into separated methods. Well-named and targeted functions not only make code more testable, but also improve readability.

Current Implementation Example In our case, the aggregation:

* Takes an input dataset (stream of words)
* Calculates word frequencies
* Produces a stream containing (word → count) pairs

The aggregation method encapsulates our core calculation logic. It could be **tested in isolation using artificial sources or sinks**. But our goal is to show an integration test.

`jubu/spark/CalculationSpec.scala`
```scala 3
package jubu.spark

import jubu.spark.Calculation._
import munit.FunSuite
import org.apache.kafka.clients.consumer.{ConsumerConfig, KafkaConsumer}
import org.apache.kafka.clients.producer.{KafkaProducer, ProducerRecord}
import org.apache.kafka.common.serialization.{StringDeserializer, StringSerializer}
import org.apache.spark.internal.Logging
import org.testcontainers.kafka.KafkaContainer

import java.io.File
import java.time.Duration
import scala.concurrent.Future
import scala.jdk.CollectionConverters._
import scala.reflect.io.Directory
import scala.util.Try

class CalculationSpec extends FunSuite with Logging {

  import scala.concurrent.ExecutionContext.Implicits.global

  private lazy val kafkaContainer = new KafkaContainer("apache/kafka:4.0.0")
    .withEnv("KAFKA_NUM_PARTITIONS", "1")
    .withEnv("AUTO_CREATE_TOPICS_ENABLE", "true")

  private def getKafkaProducer = {
    val properties = new java.util.Properties()
    properties.put("bootstrap.servers", kafkaContainer.getBootstrapServers)
    properties.put("key.serializer", classOf[StringSerializer])
    properties.put("value.serializer", classOf[StringSerializer])

    new KafkaProducer[String, String](properties)
  }

  private def getKafkaConsumer = {
    // Configuration
    val props = new java.util.Properties()
    import ConsumerConfig._
    props.put(BOOTSTRAP_SERVERS_CONFIG, kafkaContainer.getBootstrapServers)
    props.put(GROUP_ID_CONFIG, "scala-consumer-group")
    props.put(KEY_DESERIALIZER_CLASS_CONFIG, classOf[StringDeserializer])
    props.put(VALUE_DESERIALIZER_CLASS_CONFIG, classOf[StringDeserializer])
    props.put(AUTO_OFFSET_RESET_CONFIG, "earliest")
    val c = new KafkaConsumer[String, String](props)
    c.subscribe(java.util.Collections.singletonList("output-topic"))
    c
  }

  // Create a SparkSession
  private def getSpark = org.apache.spark.sql.SparkSession
    .builder()
    .appName("CalculationSpec")
    .master("local[3]")
    .config("spark.streaming.stopGracefullyOnShutdown", "true")
    .config("spark.sql.streaming.forceDeleteTempCheckpointLocation", "true")
    .getOrCreate()

  override def beforeAll(): Unit = {
    kafkaContainer.start()
  }

  override def afterAll(): Unit = {
    kafkaContainer.stop()
    Try(new Directory(new File("/tmp/checkpoint")).deleteRecursively())
  }

  val topic = "test-topic"

  test("calculation should process data from Kafka") {

    val msg        = "Ahoj, ako sa mas"
    val iterations = 100

    // create producer and consumer and spark session
    val kafkaProducer = getKafkaProducer
    val kafkaConsumer = getKafkaConsumer
    val spark         = getSpark

    // buil streaming query
    val query = build(spark, kafkaContainer.getBootstrapServers, topic).start()

    try {
      // async send an events to kafka topic
      Future {
        1 to iterations foreach { i =>
          kafkaProducer.send(new ProducerRecord[String, String](topic, i.toString, msg)).get()
          Thread.sleep(50)
        }
        kafkaProducer.close()
      }
      val res = {
        // consume events from kafka to validate
        @scala.annotation.tailrec
        def rec(time: Long, acc: Map[String, Int] = Map.empty): Boolean = {
          log.debug(s"acc: $acc")
          if (System.currentTimeMillis() - time > 15000) {
            log.error("Timeout occurred ...")
            false
          } else if (acc.values.sum != iterations * splitter(msg).size) {
            val events = kafkaConsumer.poll(Duration.ofMillis(1000)).asScala.map(r => (r.key(), r.value().toInt))
            rec(time, acc ++ events)
          } else {
            true
          }
        }
        rec(System.currentTimeMillis())
      }
      assert(res, "The streaming query completed) successfully.")
    } finally {
      // close all resources
      Try(kafkaConsumer.close())
      Try(query.stop())
      Try(query.awaitTermination())
      Try(spark.close())
    }
  }
}
```

* In the `beforeAll` method is started one Kafka instance. We use test container framework, namely: org.testcontainers:kafka (see line 33 in build.sbt)
* We create `KafkaProducer` that in `Future` sends some messages - lines 84-90. `Future` is producing data in independent thread, so our simulation is realistic.
* On line 80 we build and start our query
* Result is calculated in recursive function. We consume kafka messages from target topic and if desired number of messages is reached, function returns true else after a timeout false is returned. See lines 91-107
* Close all resources in finally block
* In `afterAll` method Kafka instance is closed

#Main

`jubu/spark/Main.scala`
```scala 3
package jubu.spark
import org.apache.spark.sql._
object Main {

  def main(args: Array[String]): Unit = {
    import Calculation._
    if (args.length < 2) {
      System.err.println(s"""
           |Usage: Main <brokers> <groupId> <topics>
           |  <brokers> is a list of one or more Kafka brokers
           |  <topics> is a list of one or more kafka topics to consume from""".stripMargin)
      System.exit(1)
    }
    val Array(brokers, topics) = args
    val spark                  = SparkSession
      .builder()
      .appName("Spark 4.0 Example1")
      .config("spark.master", "local[*]")
      .getOrCreate()
    val query = build(spark, brokers, topics).start()
    query.awaitTermination()
  }
}
```

The main method "check" arguments and then simply creates SparkSession and runs query.

#### Build commands:

* sbt assembly - build fat jar (example-1/target/scala-2.13/spark-example-1.jar)
* sbt test - run all tests
* sbt "project example-1" "run brokers topics" - run locally

Any comments or remarks are welcomed.