# Spark data sources and sinks
## Spark Data Sources and Sinks Intro
External data sources. External of your data lake, such as Oracle, SQL Server, application logs, MongoDB, Snowflake, Kafka, etc. 

You cannot process the data from external data sources directly using Spark. You could do either of these:
1. Bring the data to the data lake, and store them in your lake's distributed storage, using data integration tool. 
2. Directly connect to these external systems, using the Spark Data Source API. 

The instructor of the course prefers to use the 1st method for all their batch processing requirements, and the 2nd method for all their stream processing requirements. 

Bring data correctly and efficiently to the data lake it a complex goal by itself. Decoupling the ingestion from the processing could improve the manageability. Also, the capacity of your source system could have been planned accordingly, because it might have been designed for some specific purpose, other than this. 

Spark is an excellent tool for data processing, but it wasn't designed to handle the complexities of data ingestion. So, most of the well-designed real life projects do not directly connect to the external systems, even if we can do it. 

Internal data sources. It is your distributed storage, like HDFS, or cloud-based storage. The mechanics of reading data from these two are the same. The difference lies in the data file formats (csv, json, parquet, avro, plain text). Two other options are Spark SQL tables and Delta Lake, both are also backed by data files, but they have additional metadata stored outside of the data file. 

Data sink. The final destination of the processed data that you write to. Could be internal (a data file in your data lake storage, etc) or external (JDBC db, No sql db, etc). Spark allows you to write data in a variety of file formats, sql tables, and delta lake. 

Spark also allow you to directly write the data to many external sources, such as JDBC databases, Cassandra, MongoDB. Thought it is not recommended to do so, same reason as we don't read from these systems. 

## Spark DataFrameReader API
The "mode" option specifies behaviors when encountering a malformed record. The modes are:
- PERMISSIVE. Default. Set all the fields to null for the corrupt record, and put it in a string col named "_corrupt_record"
- DROPMALFORMED. Drop the corrupt record. 
- FAILFAST. Raises an exception, and terminates immediately, up on a malformed record. 

The "schema" is optional in many cases. Sometimes you can infer the schema. Some data formats came with a well-defined schema. 

Recommend to avoid using shortcuts such as "csv()" methods. Using the standard style add to the code maintainability. 

## Reading CSV, JSON and Parquet files
"SparkSchemaDemo.py":
```py
from pyspark.sql import SparkSession
from pyspark.sql.types import StructType, StructField, DateType, StringType, IntegerType
from lib.logger import Log4j

if __name__ == "__main__":
    spark = SparkSession \
        .builder \
        .master("local[3]") \
        .appName("SparkSchemaDemo") \
        .getOrCreate()

    logger = Log4j(spark)

    # define schema programmatically. complex. Must use spark data types
    # syntax: StructField(col_name, data_type)
    flightSchemaStruct = StructType([
        StructField("FL_DATE", DateType()),
        StructField("OP_CARRIER", StringType()),
        StructField("OP_CARRIER_FL_NUM", IntegerType()),
        StructField("ORIGIN", StringType()),
        StructField("ORIGIN_CITY_NAME", StringType()),
        StructField("DEST", StringType()),
        StructField("DEST_CITY_NAME", StringType()),
        StructField("CRS_DEP_TIME", IntegerType()),
        StructField("DEP_TIME", IntegerType()),
        StructField("WHEELS_ON", IntegerType()),
        StructField("TAXI_IN", IntegerType()),
        StructField("CRS_ARR_TIME", IntegerType()),
        StructField("ARR_TIME", IntegerType()),
        StructField("CANCELLED", IntegerType()),
        StructField("DISTANCE", IntegerType())
    ])

    # if use no schema option, all col will be str data type. 
    # if use the inferSchema option, they dates got inferred as string, 
    # but the numbers a inferred as int as expected. 
    # 
    flightTimeCsvDF = spark.read \
        .format("csv") \
        .option("header", "true") \
        .schema(flightSchemaStruct) \
        .option("mode", "FAILFAST") \
        .option("dateFormat", "M/d/y") \
        .load("data/flight*.csv") # you can use wild card here

    flightTimeCsvDF.show(5)
    logger.info("CSV Schema:" + flightTimeCsvDF.schema.simpleString())

    # the simpler/easier way to define schema
    # col_name and data_type, separated by comma. 
    flightSchemaDDL = """FL_DATE DATE, OP_CARRIER STRING, OP_CARRIER_FL_NUM INT, ORIGIN STRING, 
          ORIGIN_CITY_NAME STRING, DEST STRING, DEST_CITY_NAME STRING, CRS_DEP_TIME INT, DEP_TIME INT, 
          WHEELS_ON INT, TAXI_IN INT, CRS_ARR_TIME INT, ARR_TIME INT, CANCELLED INT, DISTANCE INT"""

    # json format doesn't have infer schema, as it already try to do it. 
    # but without doing anything specific, it read dates as strings. 
    flightTimeJsonDF = spark.read \
        .format("json") \
        .schema(flightSchemaDDL) \
        .option("dateFormat", "M/d/y") \
        .load("data/flight*.json")

    flightTimeJsonDF.show(5)
    logger.info("JSON Schema:" + flightTimeJsonDF.schema.simpleString())

    flightTimeParquetDF = spark.read \
        .format("parquet") \
        .load("data/flight*.parquet")

    flightTimeParquetDF.show(5)
    logger.info("Parquet Schema:" + flightTimeParquetDF.schema.simpleString())
```

You should prefer using the Parquet file format as long as it is possible. It is als the recommended and default file format for Apache Spark. 

Common Spark data types in Python: int, long, float, string, datetime.date, datetime.datetime, list/tuple/array, dict. 

And these data types in Spark data types: IntegerType, LongType, FloatType, DoubleType, StringType, DateType, TimestampType, ArrayType, MapType. 

## Spark DataFrameWriter API
The DataFrameWriter API allows for writing data. Its general structure:
```
DataFrameWriter
    .format(...)
    .option(...)
    .partitionBy(...)
    .bucketBy(...)
    .sortBy(...)
    .save()
```

"save mode" has 4 options:
- append. 
- overwrite. Remove existing data files, and create new files. 
- errorIfExists. 
- ignore. If data files already exist, do nothing. Otherwise write the data files. 

By default, you get one output file per partition, because each partition is written by an executor core in parallel. You can customize by re-partitioning the data. 

"partitionBy()" method re-partition your data based on a key column or a composite column key. Key-based partitioning breaks your data logically, and helps to improve your Spark SQL performance, using partition pruning. 

Bucketing with the "bucketBy()" method. Partition your data into a fixed number of predefined buckets. It is only available on Spark managed tables. 

"sortBy()" commonly used with the "bucketBy()", to create sorted buckets. 

MaxRecordsPerFile is an option. Can be used with or without "partitionBy()". Helps to control the file size based on the number of records, and protect you from creating huge/inefficient files. 

## Writing Your Data and Managing Layout


## Spark Databases and Tables


## Working with Spark SQL Tables










