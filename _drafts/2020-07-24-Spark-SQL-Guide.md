---
layout: post
title:  Spark SQL Guide
date:   2020-07-24
categories: Spark
---

- 对比
  - 相较于 basic Spark RDD API，Spark SQL 给 Spark 提供了额外的，关于数据和计算的类型信息，有利于进行运算优化
- 交互
  - 可以通过 SQL 或者 Dataset API 与 Spark SQL 进行交互
  - 不管通过什么方式交互，最终都会使用同样的执行引擎，方便用户灵活选择 API
- SQL
  - One use of Spark SQL is to execute SQL queries
- Dataset
  - A Dataset is a distributed collection of data
  - 优势
    - the benefits of RDDs (strong typing, ability to use powerful lambda functions)
    - the benefits of Spark SQL’s optimized execution engine
- DataFrame
  - A DataFrame is a Dataset organized into named columns
  - It is conceptually equivalent to a table in a relational database or a data frame in R/Python
  - In the Scala API, `DataFrame` is simply a type alias of `Dataset[Row]`

## 第一章 Getting Started

### 1. SparkSession

- The entry point into all functionality in Spark is the `SparkSession` class

- ```scala
  val spark = SparkSession
    .builder()
    .appName("Spark SQL basic example")
    .config("spark.some.config.option", "some-value")
    .getOrCreate()
  ```

### 2. 创建 DF

- 数据源

  - existing RDD
  - Hive table
  - Spark data sources
    - 在第二章详细讨论

- ```scala
  val df = spark.read.json("examples/src/main/resources/people.json")
  ```

### 3. 操作 DF

- 对比

  - untyped transformations
    - DataFrames are just Dataset of `Row`s
    - 对 DataFrame 的操作就被称为 untyped transformations
  - typed transformations
    - strongly typed Scala/Java Datasets
    - 对 Dataset 的操作就被称为 typed transformations

- 实例

  - ```scala
    // This import is needed to use the $-notation
    import spark.implicits._
    // Print the schema in a tree format
    df.printSchema()
    // root
    // |-- age: long (nullable = true)
    // |-- name: string (nullable = true)
    
    // Select only the "name" column
    df.select("name").show()
    // Select everybody, but increment the age by 1
    df.select($"name", $"age" + 1).show()
    // Select people older than 21
    df.filter($"age" > 21).show()
    // Count people by age
    df.groupBy("age").count().show()
    ```

### 4. SQL Queries

- 使用 `SparkSession` 自带的 `sql` 方法可以执行 sql 语句，返回 DataFrame

- ```scala
  // Register the DataFrame as a SQL temporary view
  df.createOrReplaceTempView("people")
  val sqlDF = spark.sql("SELECT * FROM people")
  ```

### 5. Global Temporary View

- temporary view
  - 和某个 SparkSession 绑定，生命周期在一个 session 的内部
- global temporary view
  - shared among all sessions and keep alive until the Spark application terminates
  - 注意，使用的时候需要在表名前加上 `global_temp` 库名
    - `SELECT * FROM global_temp.view1`

### 6. Creating Datasets

- 

### 7. Interoperating with RDDs

#### 1. Inferring the Schema Using Reflection

#### 2. Programmatically Specifying the Schema

### 8. Scalar Functions

### 9. Aggregate Functions