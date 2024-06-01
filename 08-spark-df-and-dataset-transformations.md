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



## Dataframe Rows and Unstructured data



## Working with Dataframe Columns



## Creating and Using UDF



## Misc Transformations


