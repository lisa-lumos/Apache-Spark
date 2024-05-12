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
# this code uses the Builder Patter in Design Patterns - 
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

## Common problem with Databricks Community


## Working with Spark SQL


## Dataframe Transformations and Actions


## Applying Transformations


## Querying Spark Dataframe


## More Dataframe Transformations

















