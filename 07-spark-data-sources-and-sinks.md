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


## Reading CSV, JSON and Parquet files


## Creating Spark DataFrame Schema


## Spark DataFrameWriter API


## Writing Your Data and Managing Layout


## Spark Databases and Tables


## Working with Spark SQL Tables










