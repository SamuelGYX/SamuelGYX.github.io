---
layout: post
title:  Structured Streaming Programming Guide
date:   2020-07-23
categories: Spark
---

## 第一章 Overview

- Structured Streaming is a scalable and fault-tolerant stream processing engine built on the Spark SQL engine.
- 特点
  - 把流数据当作静态数据分析
  - 使用 [Dataset/DataFrame API](http://spark.apache.org/docs/latest/sql-programming-guide.html) 来编写程序
  - 底层使用 Spark SQL engine 执行程序
- processing model
  - micro-batch
  - Continuous (since Spark 2.3)



## 第二章 Quick Example

- ```scala
  import org.apache.spark.sql.functions._
  import org.apache.spark.sql.SparkSession
  
  val spark = SparkSession
    .builder
    .appName("StructuredNetworkWordCount")
    .getOrCreate()
    
  import spark.implicits._
  
  // Create DataFrame representing the stream of input lines from connection to localhost:9999
  val lines = spark.readStream
    .format("socket")
    .option("host", "localhost")
    .option("port", 9999)
    .load()
  
  // Split the lines into words
  val words = lines.as[String].flatMap(_.split(" "))
  
  // Generate running word count
  val wordCounts = words.groupBy("value").count()
  
  // Start running the query that prints the running counts to the console
  val query = wordCounts.writeStream
    .outputMode("complete")
    .format("console")
    .start()
  
  query.awaitTermination()
  ```



## 第三章 Programming Model

- treat a live data stream as a table that is being continuously appended

### 1. Basic Concepts

- 数据处理流程：Input Table => Result Table => Output
- Input Table
  - Every trigger interval (say, every 1 second), new rows get appended to the Input Table
- Result Table
  - If there is new data, Spark will run an “incremental” query that combines the previous running counts with the new data to compute updated counts
- Output
  - Whenever the result table gets updated, we would want to write the changed result rows to an external sink
  - 分类
    - Complete Mode
    - Append Mode
    - Update Mode
- 补充：Many streaming systems require the user to maintain running aggregations themselves

### 2. Handling Event-time and Late Data

- Event-time
  - 数据生成时间，不同于 Spark 收到数据的时间
  - 这个参数可以很自然地通过 Input Table 中的一列来表示，方便用户进行 time window-based aggregations
- Late Data
  - 用户可以自定义如何处理迟到的数据，以及过期的数据

### 3. Fault Tolerance Semantics

- end-to-end exactly-once semantics
  - replayable sources
  - idempotent sinks



## 第四章 API using Datasets and DataFrames

- Since Spark 2.0, DataFrames and Datasets can represent static, bounded data, as well as streaming, unbounded data

### 1. Creating streaming DataFrames/Datasets

- Streaming DataFrames can be created through the `DataStreamReader` interface returned by `SparkSession.readStream()`

#### 1. Input Sources

- built-in sources
  - File source
  - Kafka source
  - Socket source (for testing)
  - Rate source (for testing)

#### 2. Schema inference and partition of streaming DataFrames/Datasets

- For ad-hoc use cases, you can reenable schema inference by setting `spark.sql.streaming.schemaInference` to `true`
- Partition discovery does occur when subdirectories that are named `/key=value/` are present and listing will automatically recurse into these directories

### 2. Operations on streaming DataFrames/Datasets

- 分类
  - untyped, SQL-like operations (e.g. `select`, `where`, `groupBy`)
  - typed RDD-like operations (e.g. `map`, `filter`, `flatMap`)

#### 1. Basic Operations - Selection, Projection, Aggregation

```scala
// Select the devices which have signal more than 10
df.select("device").where("signal > 10")      // using untyped APIs   
ds.filter(_.signal > 10).map(_.device)         // using typed APIs
```

#### 2. Window Operations on Event Time

- aggregate values are maintained for each window the event-time of a row falls into

- ```scala
  // Group the data by window and word and compute the count of each group
  val windowedCounts = words.groupBy(
    window($"timestamp", "10 minutes", "5 minutes"),
    $"word"
  ).count()
  ```

##### Handling Late Data and Watermarking

- ```scala
  val windowedCounts = words
      .withWatermark("timestamp", "10 minutes")
      .groupBy(
          window($"timestamp", "10 minutes", "5 minutes"),
          $"word")
      .count()
  ```

- 用户可以自定义保存多久的中间结果（threshold），例如十分钟

  - watermark = latest_event_time - threshold
  - event_time 早于 watermark 的数据（迟到的或者是暂存的数据）被认为是 invalid
  - 如果这样的 invalid 数据迟到，则不处理
  - 如果有聚合操作的中间数据（暂存的数据）在保存，且 time window 包含的范围早于 watermark，则这些中间数据会被处理后丢弃
    - 如果是 Update Mode，则输出更新的 row 之后丢弃
    - 如果是 Append Mode，则全部输出到 sink 后丢弃

- 补充

  - 系统只保证 valid data 一定处理
  - 不保证 invalid data 一定丢弃，可能处理，可能不处理，超时越多越有可能不会处理

#### 3. Join Operations

##### Stream-static Joins

- ```scala
  val staticDf = spark.read. ...
  val streamingDf = spark.readStream. ...
  
  streamingDf.join(staticDf, "type")          // inner equi-join with a static DF
  streamingDf.join(staticDf, "type", "right_join")  // right outer join with a static DF
  ```

##### Stream-stream Joins

###### Inner Joins with optional Watermarking

- optional
  - Watermark delays
  - Event-time range condition
    - Time range join conditions
    - Join on event-time windows

###### Outer Joins with Watermarking

- Required
- 补充
  - The outer NULL results will be generated with a delay that depends on the specified watermark delay and the time range condition.
  - if any of the two input streams being joined does not receive data for a while, the outer (both cases, left or right) output may get delayed

###### Support matrix for joins in streaming queries

- left input
- right input
- join type
  - inner
  - left outer
  - right outer
  - full outer

#### 4. Streaming Deduplication

- ```scala
  // Without watermark using guid column
  streamingDf.dropDuplicates("guid")
  
  // With watermark using guid and eventTime columns
  streamingDf
    .withWatermark("eventTime", "10 seconds")
    .dropDuplicates("guid", "eventTime")
  ```

#### 5. Policy for handling multiple watermarks

- By default, the minimum is chosen as the global watermark

#### 6. Arbitrary Stateful Operations

- For doing such sessionization, you will have to save arbitrary types of data as state, and perform arbitrary operations on the state using the data stream events in every trigger.

#### 7. Unsupported Operations

- operations that are not supported with streaming DataFrames/Datasets
  - Multiple streaming aggregations
  - Limit and take the first N rows
  - Distinct operations
  - Sorting operations are supported on streaming Datasets only after an aggregation and in Complete Output Mode.
  - Few types of outer joins
  - some Dataset methods
    - `count()`, 使用 `ds.groupBy().count()` 替代
    - `foreach()`, 使用 `ds.writeStream.foreach(...)` 替代
    - `show()`, 使用 console sink 替代

#### 8. Limitation of global watermark

- In Append mode, if a stateful operation emits rows older than current watermark plus allowed late record delay, they will be “late rows” in downstream stateful operations (as Spark uses global watermark)

### 3. Starting Streaming Queries

- 使用 `Dataset.writeStream()` 输出 result table
  - Details of the output sink
  - Output mode
  - Query name: Optionally
  - Trigger interval: Optionally
  - Checkpoint location

#### 1. Output Modes

- Append mode (default)
  - This is supported for only those queries where rows added to the Result Table is never going to change
- Complete mode
  - This is supported for aggregation queries
- Update mode

#### 2. Output Sinks

- 分类
  - File sink
    - Stores the output to a directory
  - Kafka sink
    - Stores the output to one or more topics in Kafka
  - Foreach sink
    - Runs arbitrary computation on the records in the output
  - Console sink (for debugging)
    - Prints the output to the console/stdout every time there is a trigger
  - Memory sink (for debugging)
    - The output is stored in memory as an in-memory table
- 补充
  - Note that you have to call `start()` to actually start the execution of the query
  - 该方法返回一个 `StreamingQuery` 对象，You can use this object to manage the query, which we will discuss in the next subsection

##### Using Foreach and ForeachBatch

###### ForeachBatch

- 使用

  - ```scala
    streamingDF.writeStream.foreachBatch { (batchDF: DataFrame, batchId: Long) =>
      // Transform and write batchDF 
    }.start()
    ```

- 作用

  - Reuse existing batch data sources
  - Write to multiple locations
  - Apply additional DataFrame operations

- 补充

  - `foreachBatch` provides only at-least-once write guarantees, 可以通过 `batchId` 达成 exactly-once guarantee
  - `foreachBatch` does not work with the continuous processing mode

###### Foreach

- 使用
  - extend the class `ForeachWriter`
  - divide the data writing logic into three methods: `open`, `process`, and `close`
- 注意
  - one instance is responsible for processing one partition of the data generated in a distributed manner
  - This object must be serializable
- lifecycle
  - For each partition with `partition_id`
    - For each batch/epoch of streaming data with `epoch_id`
      - Method `open(partitionId, epochId)` is called
      - If `open(…)` returns true, for each row in the partition and batch/epoch, method `process(row)` is called
      - Method `close(error)` is called with error (if any) seen while processing rows
- 补充
  - Spark does not guarantee same output for (partitionId, epochId), 需要去重请使用 `ForeachBatch`

#### 3. Triggers

- unspecified (default)
  - micro-batch mode
- Fixed interval micro-batches
  - micro-batches mode
- One-time micro-batch
  - 运行一次就自动停止
- Continuous with fixed checkpoint interval (experimental)
  - new low-latency, continuous processing mode

### 4. Managing Streaming Queries

- 常用操作

  - ```scala
    val query = df.writeStream.format("console").start()   // get the query object
    
    query.id          // get the unique identifier of the running query that persists across restarts from checkpoint data
    query.runId       // get the unique id of this run of the query, which will be generated at every start/restart
    query.name        // get the name of the auto-generated or user-specified name
    query.explain()   // print detailed explanations of the query
    query.stop()      // stop the query
    query.awaitTermination()   // block until query is terminated, with stop() or with error
    query.exception       // the exception if the query has been terminated with error
    query.recentProgress  // an array of the most recent progress updates for this query
    query.lastProgress    // the most recent progress update of this streaming query
    ```

- 多个 query

  - ```scala
    val spark: SparkSession = ...
    // You can use sparkSession.streams() to get the StreamingQueryManager
    spark.streams.active    // get the list of currently active streaming queries
    spark.streams.get(id)   // get a query object by its unique id
    spark.streams.awaitAnyTermination()   // block until any one of them terminates
    ```

### 5. Monitoring Streaming Queries

#### 1. Reading Metrics Interactively

- `streamingQuery.lastProgress()` and `streamingQuery.status()`. `lastProgress()`
  - 返回 `StreamingQueryProgress` object
- `streamingQuery.recentProgress`
  - 返回 an array of `StreamingQueryProgress` object
- `streamingQuery.status()`
  - 返回 `StreamingQueryStatus` object

#### 2. Reporting Metrics programmatically using Asynchronous APIs

- 自定义 `StreamingQueryListener` 对象
- 使用 `sparkSession.streams.attachListener()` 方法将其注册给一个 `sparkSession`
- 之后 `sparkSession` 就会自动调用对象中的回调函数

#### 3. Reporting Metrics using Dropwizard

- Spark supports reporting metrics using the Dropwizard Library
- 需要显示设置

### 6. Recovering from Failures with Checkpointing

- 存储的数据

  - progress information (i.e. range of offsets processed in each trigger)
  - running aggregates (e.g. word counts in the quick example)

- 使用

  - ```scala
    aggDF
      .writeStream
      .outputMode("complete")
      .option("checkpointLocation", "path/to/HDFS/dir")
      .format("memory")
      .start()
    ```

### 7. Recovery Semantics after Changes in a Streaming Query

- 用户在从 checkpoint restart 之前可以对程序进行一定程度的修改，但是有很多修改是不被允许的
- There are limitations on what changes in a streaming query are allowed between restarts from the same checkpoint location

## 第五章 Continuous Processing

### 1. Experimental

- 对比
  - micro-batch processing
    - exactly-once, latencies of ~100ms at best
  - continuous processing
    - at-least-once, latencies of ~1 ms
- 使用
  - specify a continuous trigger with the desired checkpoint interval
  - 完全不需要更改程序中对 dataframe 的操作
- 补充
  - The resulting checkpoints are in a format compatible with the micro-batch engine

### 2. Supported Queries

- As of Spark 2.4
  - Operations
    - Only map-like Dataset/DataFrame operations
      - rojections (`select`, `map`, `flatMap`, `mapPartitions`, etc.)
      - selections (`where`, `filter`, etc.)
    - All SQL functions are supported except aggregation functions, `current_timestamp()` and `current_date()`
  - Sources
  - Sinks

### 3. Caveats

- ensure there are enough cores in the cluster to all the tasks in parallel
- Stopping a continuous processing stream may produce spurious task termination warnings
- 任务失败后需要手动从 checkpoint 重启



## 第六章 Additional Information

- Further Reading