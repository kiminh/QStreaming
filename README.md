[![Build Status](https://travis-ci.org/qiniu/QStreaming.svg?branch=master)](https://travis-ci.org/qiniu/QStreaming) [![License](https://img.shields.io/badge/License-Apache%202.0-blue.svg)](https://opensource.org/licenses/Apache-2.0) [![Gitter](https://badges.gitter.im/qiniu-streaming/community.svg)](https://gitter.im/qiniu-streaming/community?utm_source=badge&utm_medium=badge&utm_campaign=pr-badge)

QStreaming is a framework that simplifies writing and executing ETLs on top of [Apache Spark](http://spark.apache.org/)

It is based on a simple sql-like configuration file and runs on any Spark cluster

## Architecture

![architect](docs/image/architecture.png)

QStreaming is built on top of [Apache Spark](http://spark.apache.org/) and is mainly made of following components:

- **Pipeline DSL**

  A configuration file defines the queries of the ETL Pipeline, it's  made of by the input tables, metric statements, data quality check rules (optional ) and output tables

- **Pipeline DSL Parser**

  A parser  parsing the **Pipeline DSL**  using  [Antlr](https://www.antlr.org/) parser generator and build the pipeline domain models

- **Pipeline translater**

  A  translater translate the pipeline  generated by **Pipeline DSL parser** into spark transformations

- **Data Quality Checker**

  Data quality checker is use to verify/measure  intermediate or final dataset according to the data quality check rules which defined in **Pipeline DSL** file 

- **Pipeline Runner**

  Pipeline Runner scheduling and run the pipeline as a spark batch/streaming application 

## Getting started

To run QStreaming you must first define Pipeline DSL file as below.

##### Pipeline DSL

For example a simple  pipeline dsl file  should be as follows:

```sql
-- DDL for streaming input which connect to a kafka topic
-- this declares five fields based on the JSON data format.In addition, it use the ROWTIME() to declare a virtual column that generate the event time attribute from existing ts field
create stream input table user_behavior(
  user_id LONG,
  item_id LONG,
  category_id LONG,
  behavior STRING,
  ts TIMESTAMP,
  eventTime as ROWTIME(ts,'1 minutes')
) using kafka(
  kafka.bootstrap.servers="localhost:localhost:9091",
  startingOffsets=earliest,
  subscribe="user_behavior",
  "group-id"="user_behavior"
);

-- DDL for streaming output which connect to a kafka topic
create stream output table behavior_cnt_per_hour
using kafka(
  kafka.bootstrap.servers="localhost:9091",
  topic="behavior_cnt_per_hour"
)TBLPROPERTIES(
  "update-mode"="update",
  checkpointLocation = "behavior_cnt_per_hour"
);

-- CREATE VIEW count the number of "buy" records in each hour window.
create view v_behavior_cnt_per_hour as
SELECT
   window(eventTime, "1 minutes") as window,
   COUNT(*) as behavior_cnt,
   behavior
FROM user_behavior
GROUP BY
  window(eventTime, "1 minutes"),
  behavior;


--  persist metric to kafka
insert into behavior_cnt_per_hour
select
   from_unixtime(cast(window.start as LONG)/1000,'yyyy-MM-dd HH:mm') as time,
   behavior_cnt,
   behavior
from
  v_behavior_cnt_per_hour;
```

#### Run QStreaming

There are three options to run QStreaming, first to get the latest released JAR from  [here](https://github.com/qiniu/QStreaming/releases)

##### Run on a yarn cluster

To run on a cluster requires [Apache Spark](https://spark.apache.org/) v2.2+

- Run the following command:

``` bash
$SPARK_HOME/bin/spark-submit
--class com.qiniu.stream.core.Streaming \
--master yarn \
--deploy-mode client \
stream-standalone-0.0.3-jar-with-dependencies.jar  \
-j pipeline.dsl
```

##### Run on a standalone cluster

To run on a standalone cluster you must first [start a spark standalone cluster](https://spark.apache.org/docs/latest/spark-standalone.html) , and then  run the  following  command:

```bash
$SPARK_HOME/bin/spark-submit
--class com.qiniu.stream.core.Streaming \
--master spark://IP:PORT \
stream-standalone-0.0.3-jar-with-dependencies.jar \
-j pipeline.dsl
```

##### Run as a library

It's also possible to use QStreaming inside your own project

To use it adds the dependency to your project

- maven

  ```
  <dependency>
    <groupId>com.qiniu</groupId>
    <dependency>stream-core</dependency>
    <version>0.0.3</version>
  </dependency>
  ```

- gradle

  ```
  compile 'com.qiniu:stream-core:0.0.3'
  ```

- sbt

  ```
  libraryDependencies += "com.qiniu" % "stream-core" % "0.0.3"
  ```

## Datasources

we support following datasource as input:

- [Kafka](http://kafka.apache.org/) (streaming) with `json/regex/csv/avro`   format
- HDFS/S3 with `csv/json/text/parquet/avro`   storage format
- [Jdbc datasource](https://en.wikipedia.org/wiki/Java_Database_Connectivity)
- MongoDB
- [Apache Hbase](http://hbase.apache.org/)

and following datasources as output:

- [Kafka](http://kafka.apache.org/)
- [Elasticsearch](https://www.elastic.co/elasticsearch/)
- [Apache Hbase](http://hbase.apache.org/)
- MongoDB
- [Jdbc datasource](https://en.wikipedia.org/wiki/Java_Database_Connectivity)
- HDFS/S3 with `csv/json/text/parquet/avro`   storage format

## Features

### DDL Support for streaming process

```sql
create stream input table user_behavior(
  user_id LONG,
  item_id LONG,
  category_id LONG,
  behavior STRING,
  ts TIMESTAMP,
  eventTime as ROWTIME(ts,'1 minutes')
) using kafka(
  kafka.bootstrap.servers="localhost:9091",
  startingOffsets=earliest,
  subscribe="user_behavior",
  "group-id"="user_behavior"
);
```

Above DDL statement define an input which connect to a kafka topic.

For detail information  please refer to [CreateSourceTableStatement](https://github.com/qiniu/QStreaming/blob/master/stream-core/src/main/antlr4/com/qiniu/stream/core/parser/Sql.g4#L122)  for how to define an input and [CreateSinkTableStatement](https://github.com/qiniu/QStreaming/blob/master/stream-core/src/main/antlr4/com/qiniu/stream/core/parser/Sql.g4#L127) for how to define an output.

### Watermark support in sql

QStreaming supports watermark which helps a stream processing engine to deal with late data.

There are two ways to use watermark for a stream processing engine

- Adding ***ROWTIME(eventTimeField,delayThreshold)*** as a schema property in a  ddl statement

  ```sql
  create stream input table user_behavior(
    user_id LONG,
    item_id LONG,
    category_id LONG,
    behavior STRING,
    ts TIMESTAMP,
    eventTime as ROWTIME(ts,'1 minutes')
  ) using kafka(
    kafka.bootstrap.servers="localhost:9091",
    startingOffsets=earliest,
    subscribe="user_behavior",
    "group-id"="user_behavior"
  );
  ```

  Above example  means use `eventTime` as event time field  with 5 minutes delay thresholds

- Adding   ***waterMark("eventTimeField, delayThreshold")***  as a  view property in  a view statement

  ```sql
  create view v_behavior_cnt_per_hour(waterMark = "eventTime, 1 minutes") as
  SELECT
     window(eventTime, "1 minutes") as window,
     COUNT(*) as behavior_cnt,
     behavior
  FROM user_behavior
  GROUP BY
    window(eventTime, "1 minutes"),
    behavior;
  ```

Above  example  define a watermark use `eventTime` field with 1 minute threshold

### Dynamic user define function

```
-- define UDF named hello
def hello(name:String) = {
   s"hello ${name}"
};

```

QStreaming allow to define a dynamic UDF inside job.dsl, for more detail information please refer to [createFunctionStatement](https://github.com/qiniu/QStreaming/blob/master/stream-core/src/main/antlr4/com/qiniu/stream/core/parser/Sql.g4#L100)

Above example define UDF with a string parameter.

### The multiple sink for streaming application

```sql
    create stream output table output using hbase(
        quorum = 'test1:2181,test2:2181,test3:2181',
        tableName = 'buy_cnt_per_hour',
        rowKey = '<hour_of_day>',
        cf = 'cf',
        fields = '[{"qualified":"buy_cnt","value":"behavior_cnt","type":"LongType"}]',
        where = 'behavior="buy"'
    ),hbase(
        quorum = 'test1:2181,test2:2181,test3:2181',
        tableName = 'order_cnt_per_hour
        rowKey = '<hour_of_day>',
        cf = 'cf',
        fields = '[{"qualified":"order_cnt","value":"behavior_cnt","type":"LongType"}]',
        where = 'behavior="order"'
    ) TBLPROPERTIES (outputMode = update,checkpointLocation = "behavior_output");
```

QStreaming allow you to define multiple output for streaming/batch process engine by leavarage  [foreEachBatch](https://spark.apache.org/docs/latest/structured-streaming-programming-guide.html#using-foreach-and-foreachbatch) mode (only avaliable in spark>=2.4.0)

Above example will sink the behavior count metric to two hbase table, for more information about how to create multiple sink please refer to [createSinkTableStatement](https://github.com/qiniu/QStreaming/blob/master/stream-core/src/main/antlr4/com/qiniu/stream/core/parser/Sql.g4#L127)

### Variable interpolation

```sql
create batch input table raw_log
USING parquet(path="hdfs://cluster1/logs/day=<day>/hour=<hour>");
```

job.dsl file support variable interpolation from command line arguments , this  is  useful  for running QStreaming as a periodic job.

For example, you can pass the value for  `theDayThatRunAJob` and `theHourThatRunAJob` from an  [Airflow](http://airflow.apache.org/) DAG

```bash
$SPARK_HOME/bin/spark-submit
--name {{.dir}} \
--class com.qiniu.stream.core.Streaming \
--master yarn \
--deploy-mode client \
--conf spark.executor.extraClassPath=./ \
stream-standalone-0.0.3-jar-with-dependencies.jar \
-j pipeline.dsl \
-c stream.template.vars.day=theDayThatRunAJob \
-c stream.template.vars.hour=theHourThatRunAJob
```

### Kafka lag monitor

QStreaming allow to monitor the kafka topic offset lag by  adding the ***"group-id"*** connector property in  ddl statement as below

```sql
create stream input table user_behavior(
  user_id LONG,
  item_id LONG,
  category_id LONG,
  behavior STRING,
  ts TIMESTAMP,
  eventTime as ROWTIME(ts,'1 minutes')
) using kafka(
  kafka.bootstrap.servers="localhost:9091",
  startingOffsets=earliest,
  subscribe="user_behavior",
  "group-id"="user_behavior"
);
```

### Data Quality Check ###

The purpose is to "unit-test" data to find errors early, before the data gets fed to any storage.

For example, we test for the following properties of data :

- there are 5 rows in total
- values of the `id` attribute are never NULL and unique
- values of the `productName` attribute are never NULL
- the `priority` attribute can only contain "high" or "low" as value
- `numViews` should not contain negative values
- at least half of the values in `description` should contain a url
- the median of `numViews` should be less than or equal to 10

In DSL this looks as follows:

```sql 
CREATE TEST testName(testLevel=Error,testOutput=testResult) on dataset WITH 
   numRows()=5 and 
   isNotNull(id) and 
   isUnique(id) and 
   isComplete(productName) and 
   isContainedIn(priority, ["high", "low"]) and 
   isNonNegative(numViews)  and 
   containsUrl(description) >= 0.5 and 
   hasApproxQuantile（numViews, 0.5) <= 10
```



## [Contributing](https://github.com/qiniu/QStreaming/CONTRIBUTING.md)

We welcome all kinds of contribution, including bug reports, feature requests, documentation improvements, UI refinements, etc.

Thanks to all [contributors](https://github.com/qiniu/QStreaming/graphs/contributors)!!


## License
See the [LICENSE file](https://github.com/qiniu/QStreaming/LICENSE) for license rights and limitations (Apache License).



## Join QStreaming 

Join [Gitter room](https://gitter.im/qiniu-streaming/community)

