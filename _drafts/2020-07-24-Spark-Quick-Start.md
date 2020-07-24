---
layout: post
title:  Spark Quick Start
date:   2020-07-24
categories: Spark
---

- 转变
  - before Spark 2.0
    - the main programming interface of Spark was the Resilient Distributed Dataset (RDD)
  - After Spark 2.0
    - 主要使用 Dataset, which is strongly-typed like an RDD, but with richer optimizations under the hood

## 第一章 Security

- Security in Spark is OFF by default
- 所以使用 Spark 默认存在安全隐患

## 第二章 Interactive Analysis with the Spark Shell

### 1. Basics

- Dataset

  - Spark’s primary abstraction is a distributed collection of items called a Dataset

- 创建

  - ```scala
    scala> val textFile = spark.read.textFile("README.md")
    textFile: org.apache.spark.sql.Dataset[String] = [value: string]
    ```

- 操作

  - ```scala
    textFile.count() // Number of items in this Dataset
    textFile.first() // First item in this Dataset
    val linesWithSpark = textFile.filter(line => line.contains("Spark"))
    textFile.filter(line => line.contains("Spark")).count() // How many lines contain "Spark"?
    ```

### 2. More on Dataset Operations

- find the line with the most words

  - ```scala
    scala> textFile.map(line => line.split(" ").size).reduce((a, b) => if (a > b) a else b)
    res5: Int = 15
    ```

- MapReduce flow

  - ```scala
    val wordCounts = textFile.flatMap(line => line.split(" ")).groupByKey(identity).count()
    wordCounts: org.apache.spark.sql.Dataset[(String, Long)] = [value: string, count(1): bigint]
    ```

### 3. Caching

- cluster-wide in-memory cache

  - ```scala
    linesWithSpark.cache()
    ```

## 第三章 Self-Contained Applications

- 程序源代码

  - ```scala
    /* SimpleApp.scala */
    import org.apache.spark.sql.SparkSession
    
    object SimpleApp {
      def main(args: Array[String]) {
        val logFile = "YOUR_SPARK_HOME/README.md" // Should be some file on your system
        val spark = SparkSession.builder.appName("Simple Application").getOrCreate()
        val logData = spark.read.textFile(logFile).cache()
        val numAs = logData.filter(line => line.contains("a")).count()
        val numBs = logData.filter(line => line.contains("b")).count()
        println(s"Lines with a: $numAs, Lines with b: $numBs")
        spark.stop()
      }
    }
    ```

- sbt configuration file

  - ```scala
    name := "Simple Project"
    version := "1.0"
    scalaVersion := "2.12.10"
    libraryDependencies += "org.apache.spark" %% "spark-sql" % "3.0.0"
    ```

- 运行

  - 打包

    - ```shell
      # Package a jar containing your application
      $ sbt package
      ```

  - 执行

    - ```shell
      # Use spark-submit to run your application
      $ YOUR_SPARK_HOME/bin/spark-submit \
        --class "SimpleApp" \
        --master local[4] \
        target/scala-2.12/simple-project_2.12-1.0.jar
      ```

## 第四章 Where to Go from Here

- SQL programming guide