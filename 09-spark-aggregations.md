# 9. Spark Aggregations
## Aggregating Dataframes
All aggregations in spark are implemented by built-in functions. `pyspark.sql.functions`. 

```py
from pyspark.sql import SparkSession
from pyspark.sql import functions as f

from lib.logger import Log4j

if __name__ == "__main__":
    spark = SparkSession \
        .builder \
        .appName("Agg Demo") \
        .master("local[2]") \
        .getOrCreate()

    logger = Log4j(spark)

    invoice_df = spark.read \
        .format("csv") \
        .option("header", "true") \
        .option("inferSchema", "true") \
        .load("data/invoices.csv")

    # summarize the entire df, using spark functions
    invoice_df.select(
        f.count("*").alias("Count *"),
        f.sum("Quantity").alias("TotalQuantity"),
        f.avg("UnitPrice").alias("AvgPrice"),
        f.countDistinct("InvoiceNo").alias("CountDistinct")
    ).show()

    # summarize the entire df, using sql
    invoice_df.selectExpr(
        "count(*) as `count 1`",  # same as count(1)
        "count(StockCode) as `count field`", # counting a field will not count nulls
        "sum(Quantity) as TotalQuantity",
        "avg(UnitPrice) as AvgPrice"
    ).show()

    # summarize the df, with group by cols, using sql
    invoice_df.createOrReplaceTempView("sales")
    summary_sql = spark.sql("""
        select Country, InvoiceNo,
            sum(Quantity) as TotalQuantity,
            round(sum(Quantity*UnitPrice),2) as InvoiceValue
        from sales
        group by Country, InvoiceNo
    """
    )

    summary_sql.show()

    # summarize the df, with group by cols, using spark functions
    # the last 2 cols are the same
    summary_df = invoice_df \
        .groupBy("Country", "InvoiceNo") \
        .agg(
            f.sum("Quantity").alias("TotalQuantity"),
            f.round(f.sum(f.expr("Quantity * UnitPrice")), 2).alias("InvoiceValue"),
            f.expr("round(sum(Quantity * UnitPrice), 2) as InvoiceValueExpr")
        )

    summary_df.show()
```

## Grouping Aggregations
"GroupingDemo.py":
```py
from pyspark.sql import SparkSession
from pyspark.sql import functions as f

from lib.logger import Log4j

if __name__ == "__main__":
    spark = SparkSession \
        .builder \
        .appName("Agg Demo") \
        .master("local[2]") \
        .getOrCreate()

    logger = Log4j(spark)

    invoice_df = spark.read \
        .format("csv") \
        .option("header", "true") \
        .option("inferSchema", "true") \
        .load("data/invoices.csv")

    # you can define the aggregation as a python variable, then use them in agg()
    NumInvoices = f.countDistinct("InvoiceNo").alias("NumInvoices")
    TotalQuantity = f.sum("Quantity").alias("TotalQuantity")
    InvoiceValue = f.expr("round(sum(Quantity * UnitPrice),2) as InvoiceValue")

    exSummary_df = invoice_df \
        .withColumn("InvoiceDate", f.to_date(f.col("InvoiceDate"), "dd-MM-yyyy H.mm")) \
        .where("year(InvoiceDate) == 2010") \
        .withColumn("WeekNumber", f.weekofyear(f.col("InvoiceDate"))) \
        .groupBy("Country", "WeekNumber") \
        .agg(NumInvoices, TotalQuantity, InvoiceValue)

    exSummary_df.coalesce(1) \
        .write \
        .format("parquet") \
        .mode("overwrite") \
        .save("output")

    exSummary_df.sort("Country", "WeekNumber").show()
```

## Windowing Aggregations






































