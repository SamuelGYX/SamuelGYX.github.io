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

- ```scala
  case class Person(name: String, age: Long)
  
  // Encoders are created for case classes
  val caseClassDS = Seq(Person("Andy", 32)).toDS()
  caseClassDS.show()
  
  // Encoders for most common types are automatically provided by importing spark.implicits._
  val primitiveDS = Seq(1, 2, 3).toDS()
  primitiveDS.map(_ + 1).collect() // Returns: Array(2, 3, 4)
  
  // DataFrames can be converted to a Dataset by providing a class. Mapping will be done by name
  val path = "examples/src/main/resources/people.json"
  val peopleDS = spark.read.json(path).as[Person]
  peopleDS.show()
  ```

### 7. Interoperating with RDDs

- two different methods for converting existing RDDs into Datasets

#### 1. Inferring the Schema Using Reflection

- The Scala interface for Spark SQL supports automatically converting an RDD containing case classes to a DataFrame

- ```scala
  val peopleDF = spark.sparkContext
    .textFile("examples/src/main/resources/people.txt")
    .map(_.split(","))
    .map(attributes => Person(attributes(0), attributes(1).trim.toInt))
    .toDF()
  ```

#### 2. Programmatically Specifying the Schema

- 步骤

  - Create an RDD of `Row`s from the original RDD
  - Create the schema represented by a `StructType` matching the structure of `Row`s in the RDD created in Step 1
  - Apply the schema to the RDD of `Row`s via `createDataFrame` method provided by `SparkSession`

- ```scala
  // Create an RDD
  val peopleRDD = spark.sparkContext.textFile("examples/src/main/resources/people.txt")
  
  // The schema is encoded in a string
  val schemaString = "name age"
  
  // Generate the schema based on the string of schema
  val fields = schemaString.split(" ")
    .map(fieldName => StructField(fieldName, StringType, nullable = true))
  val schema = StructType(fields)
  
  // Convert records of the RDD (people) to Rows
  val rowRDD = peopleRDD
    .map(_.split(","))
    .map(attributes => Row(attributes(0), attributes(1).trim))
  
  // Apply the schema to the RDD
  val peopleDF = spark.createDataFrame(rowRDD, schema)
  ```

### 8. Scalar Functions

- 标量函数
- Scalar functions are functions that return a single value per row

### 9. Aggregate Functions

- 聚合函数
- Aggregate functions are functions that return a single value on a group of rows



## 第二章 Data Source

### 1. Generic Load/Save Functions

#### 1. Manually Specifying Options

#### 2. Run SQL on files directly

#### 3. Save Modes

#### 4. Saving to Persistent Tables

#### 5. Bucketing, Sorting and Partitioning

### 2. Generic File Source Options

#### 1. Ignore Corrupt Files

#### 2. Ignore Missing Files

#### 3. Path Global Filter

#### 4. Recursive File Lookup

### 3. Parquet Files

#### 1. Loading Data Programmatically

#### 2. Partition Discovery

#### 3. Schema Merging

#### 4. Hive metastore Parquet table conversion

#### 5. Configuration

### 4. ORC Files

### 5. JSON Files

### 6. Hive Tables

#### 1. Specifying storage format for Hive tables

#### 2. Interacting with Different Versions of Hive Metastore

### 7. JDBC To Other Databases

### 8. Avro Files

#### 1. Deploying

#### 2. Load and Save Functions

#### 3. to_avro() and from_avro()

#### 4. Data Source Option

#### 5. Configuration

#### 6. Compatibility with Databricks spark-avro

#### 7. Supported types for Avro -> Spark SQL conversion

#### 8. Supported types for Spark SQL -> Avro conversion

### 9. Whole Binary Files

### 10. Troubleshooting

