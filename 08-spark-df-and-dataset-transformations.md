# Spark Dataframe and Dataset Transformations
## Introduction to Data Transformation
After reading the data, the dataframe is the programmatic interface for your data, and the database table is the SQL interface of your data. 

Transformations:
- combining dataframes. join/union. 
- agg and summarizing. grouping/windowing/rollups. 
- applying functions, built-in transformations. filtering/sorting/splitting/sampling/finding-uniques. 
- creating/using built-in functions, UDFs
- referencing rows/cols
- creating col expressions

## Working with Dataframe Rows
Spark notebook:
```py
# cell
from pyspark.sql import *
from pyspark.sql.functions import *
from pyspark.sql.types import *

def to_date_df(df, format, field): # a function that convert a sting col to date 
    return df.withColumn(field, to_date(col(field), format))
    #                    field is string of col name

# cell
my_schema = StructType([
    StructField("ID", StringType()),
    StructField("EventDate", StringType())
])

my_rows = [
    Row("123", "04/05/2020"), 
    Row("124", "4/5/2020"), 
    Row("125", "04/5/2020"), 
    Row("126", "4/05/2020")
]
my_rdd = spark.sparkContext.parallelize(my_rows, 2)
my_df = spark.createDataFrame(my_rdd, my_schema)

# cell
my_df.printSchema()
my_df.show()

new_df = to_date_df(my_df,  "M/d/y", "EventDate")
new_df.printSchema()
new_df.show() 
# +---+----------+ 
# | ID| EventDate| 
# +---+----------+ 
# |123|2020-04-05| 
# |124|2020-04-05| 
# |125|2020-04-05| 
# |126|2020-04-05| 
# +---+----------+
```

## DataFrame Rows and Unit Testing
Convert the prv script into an automated unit test. 

"RowDemo.py":
```py
from pyspark.sql import *
from pyspark.sql.functions import *
from pyspark.sql.types import *
from lib.logger import Log4j


def to_date_df(df, format, field):
    return df.withColumn(field, to_date(col(field), format))

if __name__ == "__main__":
    spark = SparkSession \
        .builder \
        .master("local[3]") \
        .appName("RowDemo") \
        .getOrCreate()

    logger = Log4j(spark)

    my_schema = StructType([
        StructField("ID", StringType()),
        StructField("EventDate", StringType())])

    my_rows = [Row("123", "04/05/2020"), Row("124", "4/5/2020"), Row("125", "04/5/2020"), Row("126", "4/05/2020")]
    my_rdd = spark.sparkContext.parallelize(my_rows, 2)
    my_df = spark.createDataFrame(my_rdd, my_schema)

    my_df.printSchema()
    my_df.show()
    new_df = to_date_df(my_df, "M/d/y", "EventDate")
    new_df.printSchema()
    new_df.show()
```

"RowDemo_Test.py":
```py
from datetime import date
from unittest import TestCase
from pyspark.sql import *
from pyspark.sql.types import *
from RowDemo import to_date_df

class RowDemoTestCase(TestCase):
    @classmethod
    def setUpClass(cls) -> None:
        cls.spark = SparkSession.builder \
            .master("local[3]") \
            .appName("RowDemoTest") \
            .getOrCreate()

        my_schema = StructType([
            StructField("ID", StringType()),
            StructField("EventDate", StringType())])

        my_rows = [Row("123", "04/05/2020"), Row("124", "4/5/2020"), Row("125", "04/5/2020"), Row("126", "4/05/2020")]
        my_rdd = cls.spark.sparkContext.parallelize(my_rows, 2)
        cls.my_df = cls.spark.createDataFrame(my_rdd, my_schema)

    def test_data_type(self):
        # collect() brings the actual data from the executors to the driver
        rows = to_date_df(self.my_df, "M/d/y", "EventDate").collect()
        for row in rows:
            self.assertIsInstance(row["EventDate"], date)

    def test_date_value(self):
        rows = to_date_df(self.my_df, "M/d/y", "EventDate").collect()
        for row in rows:
            self.assertEqual(row["EventDate"], date(2020, 4, 5))
```

## Dataframe Rows and Unstructured data
Assume you have a text file of logs. You read it into a df, all you get is a col of strings. You need to create new cols from it. 

"LogFIleDemo.py":
```py
from pyspark.sql import *
from pyspark.sql.functions import regexp_extract, substring_index

if __name__ == "__main__":
    spark = SparkSession \
        .builder \
        .master("local[3]") \
        .appName("LogFileDemo") \
        .getOrCreate()

    file_df = spark.read.text("data/apache_logs.txt")
    file_df.printSchema() # get a "value" col of string type

    # the regex that matches each row, to extract fields
    log_reg = r'^(\S+) (\S+) (\S+) \[([\w:/]+\s[+\-]\d{4})\] "(\S+) (\S+) (\S+)" (\d{3}) (\S+) "(\S+)" "([^"]*)'

    # create a new df with 4 cols
    logs_df = file_df.select(
        regexp_extract('value', log_reg, 1).alias('ip'),
        regexp_extract('value', log_reg, 4).alias('date'),
        regexp_extract('value', log_reg, 6).alias('request'),
        regexp_extract('value', log_reg, 10).alias('referrer')
    )

    # now have a schema, so can do analysis
    logs_df \
        .where("trim(referrer) != '-' ") \
        .withColumn("referrer", substring_index("referrer", "/", 3)) \
        .groupBy("referrer") \
        .count() \
        .show(100, truncate=False)
```

## Working with Dataframe Columns
Most of the df transformations are about transforming the columns. 

Spark databricks notebook:
```py
# cell
# check the sample datasets provided by databricks
%fs ls /databricks-datasets/

# cell
%fs ls /databricks-datasets/airlines/

# cell
# see the csv data of one file
%fs head /databricks-datasets/airlines/part-00000

# cell
airlinesDF = spark.read \
    .format("csv") \
    .option("header", "true") \
    .option("inferSchema","true") \
    .option("samplingRatio", "0.0001") \
    .load("/databricks-datasets/airlines/part-00000")

# Spark df columns are Column type objects
# the select() method accepts column strings or column objects. 

# cell
# access a column using "column strings"
airlinesDF.select("Origin", "Dest", "Distance" ).show(10)

# cell
# access a column using "column objects"
# column("Origin") uses the column() function
# col("Dest") uses the col() function, which is shorthand of column() function
# all 4 below col selections are the same
from pyspark.sql.functions import *
airlinesDF.select(
    column("col_name1"), 
    col("col_name2"), 
    "col_name3", 
    airlinesDF.col_name4
).show(10)
     
# cell
airlinesDF.select(
    "Origin", 
    "Dest", 
    "Distance", 
    "Year",
    "Month",
    "DayofMonth"
).show(10)

# cell
# use expr() function to convert an expression to a column object
# this method is preferred by most people
airlinesDF.select(
  "Origin", 
  "Dest", 
  "Distance", 
  expr("to_date(concat(Year, Month, DayofMonth), 'yyyyMMdd') as FlightDate")
).show(10)

# cell
# use column objects. used less common
airlinesDF.select(
    "Origin", 
    "Dest", 
    "Distance", 
    to_date(concat("Year", "Month", "DayofMonth"), "yyyyMMdd").alias("FlightDate")
).show(10)

```

## Creating and Using UDF
"UDFDemo.py":
```py
import re
from pyspark.sql import *
from pyspark.sql.functions import *
from pyspark.sql.types import *

from lib.logger import Log4j


def parse_gender(gender):
    female_pattern = r"^f$|f.m|w.m"
    male_pattern = r"^m$|ma|m.l"
    if re.search(female_pattern, gender.lower()):
        return "Female"
    elif re.search(male_pattern, gender.lower()):
        return "Male"
    else:
        return "Unknown"


if __name__ == "__main__":
    spark = SparkSession \
        .builder \
        .appName("UDF Demo") \
        .master("local[2]") \
        .getOrCreate()

    logger = Log4j(spark)

    survey_df = spark.read \
        .option("header", "true") \
        .option("inferSchema", "true") \
        .csv("data/survey.csv")

    survey_df.show(10)

    # method 1: use a UDF in a column object expression
    # register the function with the driver, and make it a dataframe UDF
    parse_gender_udf = udf(parse_gender, returnType=StringType())
    logger.info("Catalog Entry:")
    [logger.info(r) for r in spark.catalog.listFunctions() if "parse_gender" in r.name]
    # withColumn() allow you to transform or add a single column, 
    # without impacting other columns in the df
    survey_df2 = survey_df.withColumn("Gender", parse_gender_udf("Gender"))
    survey_df2.show(10)

    # method 2: use a UDF as a SQL function
    # register the UDF to the catalog
    spark.udf.register("parse_gender_udf", parse_gender, StringType())
    logger.info("Catalog Entry:")
    [logger.info(r) for r in spark.catalog.listFunctions() if "parse_gender" in r.name]
    # note the expr() here
    survey_df3 = survey_df.withColumn("Gender", expr("parse_gender_udf(Gender)"))
    survey_df3.show(10)
```

## Misc Transformations
"MiscDemo.py":
```py
from pyspark.sql import SparkSession
from pyspark.sql.functions import col, monotonically_increasing_id, when, expr
from pyspark.sql.types import *

from lib.logger import Log4j

if __name__ == "__main__":
    spark = SparkSession \
        .builder \
        .appName("Misc Demo") \
        .master("local[2]") \
        .getOrCreate()

    logger = Log4j(spark)

    # quick an dirty way to create a df
    data_list = [("Ravi", "28", "1", "2002"),
                 ("Abdul", "23", "5", "81"),  # 1981
                 ("John", "12", "12", "6"),  # 2006
                 ("Rosy", "7", "8", "63"),  # 1963
                 ("Abdul", "23", "5", "81")]  # 1981

    # toDF() attaches a schema to the df
    raw_df = spark.createDataFrame(data_list).toDF("name", "day", "month", "year").repartition(3)
    raw_df.printSchema()

    # the "id" col will be new, because it doesn't already exist in df
    # also set the id col as mono inc 
    final_df = raw_df.withColumn("id", monotonically_increasing_id()) \
        # cast to a different data type
        .withColumn("day", col("day").cast(IntegerType())) \
        .withColumn("month", col("month").cast(IntegerType())) \
        .withColumn("year", col("year").cast(IntegerType())) \
        # can also use sql here. 
        .withColumn("year", when(col("year") < 20, col("year") + 2000)
                    .when(col("year") < 100, col("year") + 1900)
                    .otherwise(col("year"))) \
        .withColumn("dob", expr("to_date(concat(day,'/',month,'/',year), 'd/M/y')")) \
        .drop("day", "month", "year") \
        .dropDuplicates(["name", "dob"]) \
        # .sort(expr("dob desc"))
        .sort(col("dob").desc())

    final_df.show()
```
