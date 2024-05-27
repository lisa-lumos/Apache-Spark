# Spark Structured API
## Intro to Spark APIs
Spark had the goal of simplifying/improving the Hadoop Map/Reduce programming model, so it came up with the idea of RDD (Resilient Distributed Dataset). It also came up with higher level APIs, such as Dataset APIs and Dataframe APIs. Also Spark SQL, and Catalyst Optimizer. 

The Spark community doesn't recommend using the RDD APIs. The Catalyst Optimizer decides how the code is executed, and lays out an execution plan. 

Spark SQL is the most convenient option. Use it wherever applicable. However, it lacks debugging, application logs, unit testing, etc, that a programming language could provide. 

So, a sophisticated data pipeline push you to use DataFrame APIs.

The Dataset APIs are the language-native APIs in Scala and Java. So they are NOT available in dynamically-typed languages, such as Python. 

## Intro to Spark RDD API
RDD is internally broken down into partitions to form a distributed collection. They are similar to DataFrames, but lack a row/col structure and schema. 

RDD is fault-tolerant, because they also store information about how they are created, in case the executor fails/crashes. When it happens, the drive will notice it, and assign the same RDD partition to another executor. 

## Working with Spark SQL
Spark SQL is as performant as the data frames. 

"HelloSparkSQL.py":
```py
import sys
from pyspark.sql import SparkSession
from lib.logger import Log4j

if __name__ == "__main__":
    spark = SparkSession \
        .builder \
        .master("local[3]") \
        .appName("HelloSparkSQL") \
        .getOrCreate()

    logger = Log4j(spark)

    if len(sys.argv) != 2:
        logger.error("Usage: HelloSpark <filename>")
        sys.exit(-1)

    surveyDF = spark.read \
        .option("header", "true") \
        .option("inferSchema", "true") \
        .csv(sys.argv[1])

    surveyDF.createOrReplaceTempView("survey_tbl")
    countDF = spark.sql("select Country, count(1) as count from survey_tbl where Age<40 group by Country")

    countDF.show()
```

## Spark SQL Engine and Catalyst Optimizer














