# 3. Getting Started with Apache Spark
## Introduction to Spark Data Frames
Spark allow you to choose the file format, and it supports csv, json, parquet, avro, xml, etc. 

Spark dataframe is same as a table without a metadata store (schema info). The schema info stores in the runtime metadata catalog, because the Spark dataframe is a runtime (temporary) object, and it supports schema on read. 

You can use SQL on table, but Dataframe doesn't support SQL expressions - you must use dataframe APIs to process data from a dataframe. 

You can use SQL and dataframe API to process data with Spark. 

## Creating Spark Dataframe
```py
# create a spark dataframe
# the spark in the 1st line is a SparkSession object, 
# which is the entry point for the spark programming APIs. 
# the read is an attribute of the spark session object, so no parenthesis
# it returns a DataFrameReader object
# the format(), option() and load() are methods of the DataFrameReader. 
# this code uses the Builder Pattern in Design Patterns - 
# each method call represents one step in the DataFrameReader. 
# It finally returns a DataFrame object
# refer to:
# spark.apache.org/docs/latest/api/python/index.html
# -> API Reference 
# -> Spark SQL (most Spark dataframe APIs are grouped under this package)
fire_df = spark.read \ 
    .format("csv") \
    .option("header", "true") \
    .option("inferSchema", "true") \
    .load("/databricks-datasets/learning-spark-v2/sf-fire/sf-file-calls.csv")

# show the first 10 records of the df
fire_df.show(10)

# show the df in a tabular format
# This is a tool provided by Databricks only
display(fire_df)

# create a global temporary view for the df, 
# view name is fire_service_calls_view
# view is created in the hidden database "global_temp"
# it is where all temporary tables are created
fire_df.createGlobalTempView("fire_service_calls_view")
```

Then you can query this view, using a sql cell: 
```sql
select * 
from global_temp.fire_service_calls_view
```

## Creating Spark Tables
Create a new notebook, set default language as SQL. 
```sql
-- cell 1. 
-- can skip semicolon if the cell only contain one command
-- Run it, and can see it in the data pane in the left
create database if not exists demo_db

-- cell 2
-- every db table internally stores data in files
-- and this table's data file is stored in parquet format
create table if not exists demo_db.fire_service_calls_tbl(
  CallNumber integer,
  UnitID string,
  ... -- skipped more cols
) using parquet

-- cell 3
-- can insert using sql
-- but spark tables are meant to hold large amount of data
-- so usually this is not how data is loaded into spark tables
insert into demo_db.fire_service_calls_tbl values (
  1234, null, null, ...
)

-- cell 4
select * from demo_db.fire_service_calls_tbl

-- cell 5
-- spark sql doesn't offer delete statements
truncate table demo_db.fire_service_calls_tbl

-- cell 6
-- insert into spark table using a view that was previously created
insert into demo_db.fire_service_calls_tbl
select * from global_temp.fire_service_calls_view

```

Documentation location: spark docs -> Latest Release -> Programming Guides -> SQL, DataFrames, and Datasets -> SQL Reference -> SQL Syntax -> Data Definition Statements

## Common problem with Databricks Community Edition
When a cluster terminates, both the cluster VM and the Spark Metadata store is gone. So you will lose you databases and tables. This problem doesn't exist for the full licensed version of the Databricks Cloud. 

The community edition will not clean up the directories and files. So you do not lose your storage layer. This external storage is not part of your cluster. 

Manually cleaning up the data files with a new cell `%fs rm -r /user/hive/warehouse/demo_db.db` (do not delete the warehouse dir, it is the default location for all Spark databases), then in a new cell, do `drop table if exists demo_db.fire_service_calls_tbl; drop database if exits demo_db;`, and running the entire notebook with a new cluster will re-create the metadata layer. 

## Working with Spark SQL
skipped.

## Dataframe Transformations and Actions
Documentation -> Latest Release -> API Docs -> Python -> API Reference -> Spark SQL -> pyspark.sql.DataFrame

Dataframe method types:
- Actions. Trigger a Spark job, and return to the Spark driver (vs executor). 
- Transformations. Produce a newly transformed dataframe. 
- Functions/Methods. Not actions nor transformations. 

Actions: collect, count (when input is a df), describe, first, foreach/foreachPartition, head/tail, show, summary, take, toLocalIterator. 

Transformations: agg, count (when input is a GroupedData object), alias, coalesce, colRegex, filter, groupby, dropna, join, limit, orderBy, select, sort, union, where, ...

Functions/Methods: cache, checkpoint, createTempView, explain, toDF, toJSON, writeTo, ...

## Applying Transformations
```python
# cell 1
raw_fire_df = spark.read \ 
    .format("csv") \
    .option("header", "true") \
    .option("inferSchema", "true") \
    .load("/databricks-datasets/learning-spark-v2/sf-fire/sf-file-calls.csv")

# cell 2
display(raw_file_df)

# cell 3
# spark dataframe column names are case-insensitive. 
# spark dataframe is immutable - you cannot change the existing dataframe, 
# instead, you transform the existing df, and create a new df. 
renamed_fire_df = raw_fire_df \
    .withColumnRenamed("Call Number", "CallNumber") # old col name, new col name
    .withColumnRenamed("Unit ID", "UnitID") # you can chain them

# cell 4
# see the schema of the df
renamed_fire_df.printSchema()

# cell 5
from pyspark.sql.functions import *

# cell 6
# withColumn(name_of_the_new_col, calculation_for_the_new_col)
fire_df = renamed_fire_df \
    .withColumn("CallDate", to_date("CallDate", "MM/dd/yyyy")) \ 
    .withColumn("AvailableDtTm", to_timestamp("AvailableDtTm", "MM/dd/yyyy hh:mm:ss a")) \
    .withColumn("Delay", round("Delay", 2))

```

## Querying Spark Dataframe
```py
# cell
# will run many analysis on this df, cache it in memory will speed it up
fire_df.cache()

# cell
# create a temp view of the df, and query the view using sql
# spark.sql() executes the sql, and returns the result into a new df
fire_df.createOrReplaceTempView("fire_service_calls_view")
sql_df_1 = spark.sql("""
select 
    count(distinct CallType) as distinct_call_type_count
from fire_service_calls_view
where CallType is not null
"""
)
display(sql_df_1)

# cell
# where() does the filtering, the condition string is same with sql
df_1 = fire_df.where("CallType is not null") \
    .select("CallType") \
    .distinct()
# best practice: keep an action on a separate line. 
print(df_1.count()) # this is an action, because it doesn't return df, it triggers spark job execution, and returns to the spark driver

# cell
# if you want to use an expression in select(),
# you must evaluate the expression using expr()
df_2 = fire_df.where("CallType is not null") \
    .select(expr("CallType as distinct_call_type")) \
    .distinct()
df_2.show()

# cell
# select() takes a comma separated list of columns
# you can chain one and only one Action, and
# it must be the last method in the chain
df_3 = fire_df.where("Delay > 5") \
    .select("CallNumber", "Delay") \
    .show()

# cell
# you can use select first, then where, or vise versa,
# because the end result is the same,
# and the performance is also the same. This rule doesn't apply to other methods. 
# groupBy() takes a comma-separated list of cols
# the count() here is a transformation, instead of an action,
# because it doesn't apply to a df, instead,
# it takes the GroupedData object, which is returned by groupBy()
df_4 = fire_df.select("CallType") \
    .where("CallType is not null") \
    .groupBy("CallType") \
    .count() \
    .orderBy("count", ascending=False) \
    .show()

```
