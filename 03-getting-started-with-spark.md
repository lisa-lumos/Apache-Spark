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


## Common problem with Databricks Community


## Working with Spark SQL


## Dataframe Transformations and Actions


## Applying Transformations


## Querying Spark Dataframe


## More Dataframe Transformations

















