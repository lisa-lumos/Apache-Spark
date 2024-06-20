# 10. Spark dataframe joins
## Dataframe Joins and column name ambiguity
"SparkJoinDemo.py":
```py
from pyspark.sql import SparkSession

from lib.logger import Log4j

if __name__ == "__main__":
    spark = SparkSession \
        .builder \
        .appName("Spark Join Demo") \
        .master("local[3]") \
        .getOrCreate()

    logger = Log4j(spark)

    orders_list = [
        ("01", "02", 350, 1),
        ("01", "04", 580, 1),
        ("01", "07", 320, 2),
        ("02", "03", 450, 1),
        ("02", "06", 220, 1),
        ("03", "01", 195, 1),
        ("04", "09", 270, 3),
        ("04", "08", 410, 2),
        ("05", "02", 350, 1)
    ]

    order_df = spark.createDataFrame(orders_list).toDF(
        "order_id", 
        "prod_id", 
        "unit_price", 
        "qty"
    )

    product_list = [
        ("01", "Scroll Mouse", 250, 20),
        ("02", "Optical Mouse", 350, 20),
        ("03", "Wireless Mouse", 450, 50),
        ("04", "Wireless Keyboard", 580, 50),
        ("05", "Standard Keyboard", 360, 10),
        ("06", "16 GB Flash Storage", 240, 100),
        ("07", "32 GB Flash Storage", 320, 50),
        ("08", "64 GB Flash Storage", 430, 25)
    ]

    product_df = spark.createDataFrame(product_list).toDF(
        "prod_id", 
        "prod_name", 
        "list_price", 
        "qty"
    )

    product_df.show()
    order_df.show()

    # define a variable as the join expression
    join_expr = order_df.prod_id == product_df.prod_id

    product_renamed_df = product_df.withColumnRenamed("qty", "reorder_qty")

    # inner join
    order_df.join(product_renamed_df, join_expr, "inner") \
        .drop(product_renamed_df.prod_id) \
        .select("order_id", "prod_id", "prod_name", "unit_price", "list_price", "qty") \
        .show()

    # left outer join
    order_df.join(product_renamed_df, join_expr, "left") \
        .drop(product_renamed_df.prod_id) \
        .select("order_id", "prod_id", "prod_name", "unit_price", "list_price", "qty") \
        .withColumn("prod_name", expr("coalesce(prod_name, prod_id)")) \
        .withColumn("list_price", expr("coalesce(list_price, unit_price)")) \
        .sort("order_id") \
        .show()

```

When you do select *, there is no complains about the duplicated prod_id, because every df has a unique id in the catalog, and the Spark engine always works using these internal ids. However, when we explicitly select column names, if same name maps to different ids, spark will complain about the ambiguity. 

The solution to this ambiguity error in join:
1. rename the related cols before the join
2. or, drop the conflicting col after the join, before the selection. 

## Internals of Spark Join and shuffle
Spark joins is one of the most common causes for slowing down your application. 

Shuffle join. Most common. Shuffle operation divides the matched join keys evenly across all executors, both from the left table and the right table. So that part keys from both left/right table can in one executor. Tuning your join operation is all about optimizing the shuffle operation. `spark.conf.set('spark.sql.shuffle.partitions', 3)` ensures you get 3 partitions after the shuffle. 

Broadcast join. 

## Optimizing your joins
Reduce df size. Cut down the size of df as early as possible. filter early, before the join.

Parallelism. If you want to take advantage of a large cluster, you should increase the number of shuffle partitions But, if you have 500 executors, but you configured to have 400 shuffle partitions, the max parallelism is limited to 400. Further, if you have only 200 unique keys, then you can have only 200 shuffle partitions, even if you define 400 shuffle partitions, only 200 of them will have data. 

Skew. Watch out for the time taken by individual tasks, and the amount of data processed by the join task. If some tasks are taking significantly longer than other tasks, you can fix your join key, or apply some hack to break the larger partition into more than one partitions. 

Large to Large(df cannot fit into a single executor's ram). Will always be a shuffle join. 

Large to Small. Can take advantage of broadcast join. Instead of shuffle and send the large data to many executors, just sent the small df to every executor. So much less amount of data is sent over the network. 

In most of the cases, Spark will automatically use the broadcast join, when one of the df is smaller and can be broadcasted. 

But you know your data better than Spark. To enforce a broadcast join, do like this: `join_df = left_df.join(broadcast(right_df), join_expr, "inner")`. 

## Implementing Bucket Joins
Shuffle join is almost unavoidable in case of a large to large dataset join. However, in some cases, you can prepare in advance, and avoid the shuffle at the time of joining. 

If you have two datasets that you already know you are going to join in the future, then it is advisable to bucket both of your datasets using your join key. Bucketing your dataset may also require a shuffle, but this shuffle is needed only once, when you create your bucket. Once bucket is created, you can join these datasets without a shuffle, and do it as many times as you need. 

"BucketJoinDemo.py":
```py
from pyspark.sql import SparkSession

from lib.logger import Log4j

if __name__ == "__main__":
    spark = SparkSession \
        .builder \
        .appName("Bucket Join Demo") \
        .master("local[3]") \
        .enableHiveSupport() \
        .getOrCreate()

    logger = Log4j(spark)
    df1 = spark.read.json("data/d1/")
    df2 = spark.read.json("data/d2/")
    # df1.show()
    # df2.show()

    spark.sql("CREATE DATABASE IF NOT EXISTS MY_DB")
    spark.sql("USE MY_DB")

    # coalesce to a single partition first, with coalesce(1)
    # bucketBy(num_of_buckets, bucket_by_key_name)
    df1.coalesce(1).write \
        .bucketBy(3, "id") \
        .mode("overwrite") \
        .saveAsTable("MY_DB.flight_data1")

    df2.coalesce(1).write \
        .bucketBy(3, "id") \
        .mode("overwrite") \
        .saveAsTable("MY_DB.flight_data2")

    df3 = spark.read.table("MY_DB.flight_data1")
    df4 = spark.read.table("MY_DB.flight_data2")

    spark.conf.set("spark.sql.autoBroadcastJoinThreshold", -1) # disable broadcast join
    join_expr = df3.id == df4.id
    join_df = df3.join(df4, join_expr, "inner")

    join_df.collect()
    input("press a key to stop...")

```
