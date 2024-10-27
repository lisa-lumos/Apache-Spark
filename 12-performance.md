# 12. Performance
## Spark Memory Allocation
You can ask for the driver memory using below configs:
- `spark.driver.memory`. Heap memory for the spark driver JVM. Assume you asked for 1GB. 
- `spark.driver.memoryOverhead`. Used by the container process, or, any other non-JVM process within the container. Default is 10% of the 1st param, or 384MB, whichever is bigger. In this case, 1GB * 10% = 100MB < 384MB, so the value is set to 384MB.  

So, in this case, the total memory of this driver container is 1GB + 384MB. If these memory limits are violated, you will see an Out Of Memory exception. 

For each executor container, the total memory is the sum of below:
1. `spark.executor.memory`. Heap memory for JVM. Assume you asked for 8GB. 
2. `spark.executor.memoryOverhead`. Overhead memory. Default is 10% of the 1st param. Also used for network buffers, such as shuffle exchange, reading partition data from remote storage, etc. So in this case, 800MB. 
3. `spark.memory.offHeap.size`. Off heap memory. Default is 0. Enabled by `spark.memory.offHeap.enabled` config (disabled by default). Most of the spark operations and data caching are performed in the JVM heap, and they perform best using the on-heap memory. However, JVM heap is subject to garbage collection, so if you allocate a huge amount of heap memory to your executor, you might see excessive garbage collection delays. However, Spark 3.x was optimized to perform some operations in the off-heap memory, giving you the flexibility to manage memory by yourself and avoid GC delays. Spark uses the off-heap meory to extend the size of spark Executor Memory and Storage Memory. 
4. `spark.executor.pyspark.memory`. PySpark memory. Default is 0. When you use PySpark, your application may need a Python worker, who cannot use JVM heap memory. So they use off-heap overhead memory. But if you need more off-heap overhead for your Python workers, you can set the extra memory requirement using this parameter. 

So, in this case, the total memory of each executor is 8GB + 800MB. If these memory limitations are violated, you also will see an OOM exception.

But whether you can get a 8.8GB ram container or not, depends on your cluster config. If your worker node is a 6GB machine, you won't have enough physical memory to begin with. So you should check with your cluster admin for the maximum allowed value. If you use YARN Resource Manager, you should look for configs `yarn.scheduler.maximum-allocation-mb`, and `yarn.nodemanager.resource.memory-mb`. 

For example, if you use AWS c4.large instance type to set up your Spark cluster, using AWS EMR, the c4.large instance comes with 3.75GB RAM. When you launch the EMR cluster, it will start with the default value of `yarm.scheduler.maximum-allocation-mb = 1792`, so you cannot ask for a container higher than 1.792GB. 

PySpark is not a JVM process. So in this case, it consumes the memory allocated to the overhead. 

Both the heap memory and the overhead memory are critical for your Spark application. More than often, lack of enough overhead memory will cost you an OOM exception. 

## Spark Memory Management
Assume you config executor heap memory with `spark.executor.memory` to 8GB. 

The executor heap memory contains:
1. Reserved Memory. 300MB fixed. The Spark Engine itself uses it. 
2. Spark Memory. Controlled by the `spark.memory.fraction` config, default to 60%. The main executor memory pool, used for df operations and caching. You can increas it to 60% to 70%, or even more, if you are not using UDFs, custom data structures, and RDD operations. But you cannot make it 100%, because you need metadata etc in User Memory. In this case it is (8GB - 300MB) * 60%. 
3. User Memory. Used for non-dataframe operations, such as user-defined data structures, Spark internal metadata, UDFs created by the user, RDD conversion operations, RDD lineage and dependency, etc. Note that the RDD part only applies when you apply RDD operations directly in your code, otherwise it uses Spark Memory. In this case it is (8GB - 300MB) * 40%. 

The Spark Memory pool is further broken down into 2 sub-pools:
1. Storage Memory Pool. Used for cache memory for dataframes, such as when you use df cache operation. It is long-term, the df keeps being cached, aslong as the executor is running, or you want to un-cache it. If you are not dependent on cached data, you can reduce this hard limit. Controlled by config `spark.memory.storageFraction`, default to 50%. 
2. Executor Memory Pool. Used for buffer memory for performing dataframe computations. It is short lived - it is used for execution, and freed immediately after the execution is complete. 

Assume you config executor cores using `spark.executor.cores` as 4. 

So each executor will have 4 slots, available for running 4 tasks in parallel. Note that all these slots are threads within the only JVM of the node - we do not have multiple processes/JVMs here - we have a single JVM. 

In this case, we have 4 threads sharing the two memory sub-pools. Starting Spark 1.6, the "unified memory manager "was introduced, who tries to implement fair allocation among the active tasks. For example, if there are only 2 active tasks, the manager can allocate all the available executor memory among the 2 tasks, instead of to all the 4 slots. Further more, if the Executor Memory pool is fully consumed, the memory manager can also allocate Storage Memory pool, as long as free space exist. 

And vise versa, if the Storage Memory pool needs more space, the memory manager can allocate more from the Executor Memory pool, as long as free space exist. 

If both the Storage Memory pool and the Executor Memory pool are fully occupied, and the executor needs more memory, then it will try to spill some things to the disk, and make more room for itself. If you do not have things that you can spill to the disk, then you hit the OOM exception. 

In general, Spark recommends 2 or more cores per executor, but you should not go beyond 5 cores, which will cause excessive memory management overhead and contention. 

## Spark Adaptive Query Execution (AQE)
A new feature released in Apache Spark 3.0. You must enable it for it to work. 

AQE does:
- Dynamically coalescing shuffle partitions.
- Dynamically switching join strategies.
- Dynamically optimizing skew joins. 

## Spark AQE coalescing shuffle partitions
Assume you group by col A, and there are only 5 unique values in col A. The shuffle sort operation will put A = val1 rows into partition 1, and A = val2 rows into partition 2, etc, so only 5 partitions are actually needed. Also, each partition may be of different size, because different number of rows are under different values of the group key (data skew). Note that by default, the value of `spark.sql.shuffle.partitions` is 200. Note that data changes, queries are also varies, so it is almost impossible to manually set a perfect value for it. 

AQE will take care of setting the number of your shuffle partitions. During shuffle/sort, it will dynamically compute the statistics in on the exchange data, during the shuffle, and dynamically set the shuffle partitions for the next stage. It might combine two small partitions into one, so that less tasks will be needed in the next stage, and that their run time is more comparable. 

To enable AQE, use `spark.sql.adaptive.enabled` param. To tune AQE, use:
- `spark.sql.adaptive.coalescePartitions.enabled`. Default to true. 
- `spark.sql.adaptive.coalescePartitions.initialPartitionNum`. max num of partitions. 
- `spark.sql.adaptive.coalescePartitions.minPartitionNum`. 
- `spark.sql.adaptive.advisoryPartitionSizeInBytes`. Default is 64MB. 

## Spark AQE Dynamic Join Optimization
Assume you are joining two large tables, then applying a filter on table 2. Without analyzing the filter first, you might implement a sort-merge join. But up on evaluating the filter, you notice that only a few mb is from table 2. So you should use a broadcast hash join. 

Regularly, spark execution plan is created before it starts the job execution, so it wouldn't have known the size of the filtered table, unless you keep your table and column statistics up-to-date, such as the column histogram for your filter column. Alternatively, you can enable AQE, who compute the statistics on the exchange data, during the shuffle, and dynamically change the execution plan. In this case, it will skip the sort operation. 

If you use AQE, do not disable local shuffle reader `spark.sql.adaptive.localShuffleReader.enabled = true` (it is enabled by default). This is designed to further optimize the AQE broadcast join, by reducing the network traffic.  

## Handling Data Skew in Spark Joins
Assume you are joining two large tables. During the shuffle step in the sort-merge join, the data was re-partitioned by the join key. Assume one partition from the left table is much larger than the others, so the task that is responsible for that join key from both tables might need a bigger memory than the rest of the tasks, or, it might take much longer to complete. 

Spark AQE solves the problem. It could split the larger partition into more partitions, and duplicate the right table's matching partition. In this case, more tasks will handle the original larger partition. 

To turn it on:
- `spark.sql.adaptive.enabled = true`. Enables the AQE feature. 
- `spark.sql.adaptive.skewJoin.enabled = true`. Enables the skew-join optimization. 

To customize it (AQE identify a partition is skewed, when both of the below thresholds are broken):
- `spark.sql.adaptive.skewJoin.skewedPartitionFactor = 5`. Default to 5, which means, the partition is skewed if its size is larger than 5 times the median partition size. 
- `spark.sql.adaptive.skewJoin.skewedPartitionThresholdInBytes = 256MB`. Default to 256MB, which means, the partition is skewed if its size in bytes is larger than this threshold. 

## Spark Dynamic Partition Pruning
A new feature available in Spark 3.0 and above. It is enabled by default: `spark.sql.optimizer.dynamicPartitionPruning.enabled = true`.

Assume you have a large fact table, you partitioned it by date col, because you know you usually query this table on this date col. Now when you run an agg query for a certain date, Spark only reads the partitions that contains data for that date. 

Predicate pushdown. Spark will push down "where" clause filters to the scan step, and apply them as early as possible. 

Partition pruning. It will only read the partition that contains the selected date. It needs your data to be already partitioned on the filter columns. 

Assume you join this fact table with a date spine dimension table, on the "date" column, and filter on that dimension table's "year" and "month" column. Before Spark 3.0, this will cause a sort-merge join, because the filter conditions are not applied on the fact table's "date" column. 

Dynamic partition pruning. With Spark 3.0, you first broadcast the date dimension table, Spark will then take the filter condition applied to the dimension table, and inject it into your fact table as a subquery, so it can apply partition pruning on the fact table. 

It works for fact and dimension-like tables. The fact table must be partitioned, and, the you must broadcast your dimension table, for this feature to work. If your dimension table is smaller than 10MB, the broadcasting happens automatically.  

## Data Caching in Spark
Storage Memory Pool can cache dataframes. 

How to cache a df:
1. cache(). Doesn't take arguments. Uses the default MEMORY_AND_DISK storage level. Memory Deserialized 1X Replicated. 
2. persist(). Takes an optional storage level argument. 

They are both lazy transformations. They are smart enough to cache only what you access. 

Examples:
```py
df = spark.range(1, 1000000).toDF("id") \
    .repartition(10) \
    .withColumn("square", expr("id * id")) \
    .cache()

df.take(10) # cache() is an lazy transformation, so need to execute an action

df.count()
```

The "take(10)" action will bring only 1 partition into the memory, and return 10 records from the loaded partition. So that only 1 partition is cached. Note that Spark will always cache the entire partition - it will not do only a portion of the partition. 

The "count()" action will bring all the partitions into the memory, compute the count, and return it. 

Examples:
```py
df = spark.range(1, 1000000).toDF("id") \
    .repartition(10) \
    .withColumn("square", expr("id * id")) \
    .persist(StorageLevel(
            False, # useDisk (if memory is not large enough)
            True,  # useMemory (cache into memory)
            False, # useOffHeap
            True,  # deserialized (whether use deserialized format in memory)
            1      # replication=1
        )
    )

df.take(10) # cache() is an lazy transformation, so need to execute an action

df.count()
```

Spark always store data on a disk in serialized format. But when data comes into Spark memory, it must be deserialized into Java objects. The deserialized format takes more space, while the serialized format is more compact. 

If you choose to use serialized format in memory, you can save some memory, but when spark need to use that data, it need to spend some extra CPU overhead, to deserialize it. The recommendation is to keep your data deserialized and save your CPU. 

If space is not enough, the order is: memory, then off heap, then disk. 

You can also use pre-defined constants. Examples:
```py
df = spark.range(1, 1000000).toDF("id") \
    .repartition(10) \
    .withColumn("square", expr("id * id")) \
    .persist(StorageLevel=StorageLevel.MEMORY_AND_DISK)

df.take(10) # cache() is an lazy transformation, so need to execute an action

df.count()
```

Some of the pre-defined constants:
- DISK_ONLY
- MEMORY_ONLY
- MEMORY_ONLY_2 (means 2 replication, but takes a lot of memory)
- MEMORY_ONLY_3
- MEMORY_AND_DISK
- MEMORY_ONLY_SER
- MEMORY_AND_DISK_SER
- OFF_HEAP
- ...

Use `df.unpersist()` to un-cache.

When to cache, and not-to-cache? When you need to access large df multiple times, across Spark actions, consider caching your df. Make sure to config your memory accordingly. Do not cache your df, when significant portions of then do not fit in the memory. If you do not frequently reuse your df, or if your df is too small, do not cache them. 

## Repartition and Coalesce
2 ways to repartition your dataframe:
1. `repartition(numPartitions, *cols)`. Hash based partitioning. 
2. `repartitionByRange(numPartitions, *cols)`. Range of values based partitioning. Uses data sampling to determine partition ranges, so the result is not deterministic. 

repartition is a wide dependency transformation, so it will cause a shuffle/sort operation. 

repartition on a column name doesn't guarantee a uniform partitioning. 

e.g.:
```py
repartition(10)
repartition(10, 'age')
repartition(10, 'age', 'gender')
repartition('age') # the default num of partitions is determined by spark.sql.shuffle.partitions config
repartition('age', 'gender')
```

Repartition causes a shuffle/sort, which is expensive, and should be avoided is unnecessary. When should you do a repartition? 
- Dataframe reuse, and repeated column filters
- Dataframe partitions are not well distributed
- Large dataframe partitions, or skewed partitions

You should use repartitioning when you want to increase the number of partitions. But when you want to decrease the num of partitions, you should use the Coalesce method - it combines local partitions on the same worker node, to meet your target number of partitions. And hence it can reduce num of partitions faster than repartitioning. 

Coalesce doesn't cause a shuffle/sort, because it combines local partitions only. Note that it can cause skewed partitions. Try to avoid drastically decrease num of partitions, because it may lead to a OOM exception. 

## Dataframe Hints
Hints give users a way to suggest how Spark SQL to use specific approaches to generate its execution plan. Spark doesn't guarantee that it will apply the hint. 

Two types of hints:
1. Partitioning hints
   - coalesce. Used to reduce num of partitions to the desired number
   - repartition. 
   - repartition_by_range. 
   - rebalance. Used to rebalance the query result partitions, so they they are of a reasonable size. Can take col names as params. Helpful when you need to write the query result to a table, to avoid too small/big files. AQE needs to be enabled to be able to use this hint. 
2. Join hints (allow you to suggest join strategy) (ordered by priority)
   - broadcast (aka, broadcastjoin/mapjoin)
   - merge (shuffle_merge/mergejoin)
   - shuffle_hash
   - shuffle_replicat_nl(shuffle and replicate nested loop join)

Using hints in Spark sql:
```sql
select /*+ coalesce(3) */ * from t;
select /*+ repartition(3) */ * from t;

select /*+ broadcast(t1) */ * from t1
inner join t2 on t1.key = t2.key;

select /*+ merge(t1) */ * from t
inner join t2 on t1.key = t2.key;
```

df.hint() method:
```py
from pyspark.sql import SparkSession
from pyspark.sql.functions import *

if __name__ == "__main__":
    spark = SparkSession.builder.appName('Demo').master('local[3]').getOrCreate()

    df1 = spark.read.json('data/d1/') # a large df
    df2 = spark.read.json('data/d2/') # a small df

    join_df = df1.join(broadcast(df2), 'id', 'inner').hint("COALESCE", 5)
    join_df.show()
    print(f'Num of output partitions: {str(join_df.rdd.getNumPartitions())}')
```

## Broadcast Variables
They are part of the Spark low-level APIs. If you are using Spark Dataframe APIs, you are not likely to use broadcast variables. They were primarily used with the Spark low-level RDD APIs.

However, it is essential to understand the concept.

For example, you have a dict that maps each product code to a product name. You need to create a udf that takes a df and a product code, and returns the product name. 

One easy way it to keep the lookup data (the dict) in a file, load it at runtime, and share it with the UDF. 

```py
from pyspark.sql import SparkSession
from pyspark.sql.functions import *
from pyspark.sql.types import *

def my_func(code: str) -> str:
    # return prdCode.get(code) # use the lambda closure
    return bdData.value.get(code) # use the broadcast variable

if __name__ == "__main__":
    spark = SparkSession \
        .builder \
        .appName("Demo") \
        .master("local[3]") \
        .getOrCreate()

    prdCode = spark.read.csv("data/lookup.csv").rdd.collectAsMap() # a python dict

    bdData = spark.sparkContext.broadcast(prdCode) # a broadcast variable

    data_list = [("98312", "2021-01-01", "1200", "01"),
                 ("01056", "2021-01-02", "2345", "01"),
                 ("98312", "2021-02-03", "1200", "02"),
                 ("01056", "2021-02-04", "2345", "02"),
                 ("02845", "2021-02-05", "9812", "02")]
    df = spark.createDataFrame(data_list) \
        .toDF("code", "order_date", "price", "qty")

    spark.udf.register("my_udf", my_func, StringType())
    df.withColumn("Product", expr("my_udf(code)")) \
        .show()

```

If you use a closure, Spark will serialize the closure to each task, e.g., if you have 1k tasks running on a 30 node cluster, your closure is serialized 1k times. But is you use a broadcast variable, Spark will serialize it once per worker node, and cache it there for future usage, so the broadcast variable is only serialized 30 times. 

Broadcast variables are shared, immutable variables, cached on every machine in the cluster, instead of serialized with every single task. And this serialization is lazy. The broadcast variable is serialized to a worker node only if you have at least one task running on the worker node that needs to access the broadcast variable.

Note that the broadcast variable must fit into the memory of an executor. 

Spark Dataframe APIs use this same technique to implement broadcast joins.

## Accumulators
Also part of the Spark low-level APIs. If you are using Spark Dataframe APIs, you are not likely to use accumulators. They were primarily used with the Spark low-level RDD APIs. However, it is essential to understand the concept.

Spark Accumulator is a global mutable variable that a Spark cluster can safely update on a per-row basis. You can use them to implement counters or sums. You do not have to collect it from anywhere, because the accumulators always live at the driver.

Assume you need to count num of nulls in a col in a df, and decide to create a UDF for this. However, count() operation is a wide dependency action, which we try to avoid.

```py
from pyspark.sql import SparkSession
from pyspark.sql.functions import *
from pyspark.sql.types import *


def count_nulls(shipments: str) -> int:
    s = None
    try:
        s = int(shipments)
    except ValueError:
        n_nulls.add(1)
    return s


if __name__ == "__main__":
    spark = SparkSession \
        .builder \
        .appName("Demo") \
        .master("local[3]") \
        .getOrCreate()

    data_list = [("india", "india", '5'),
                 ("india", "china", '7'),
                 ("china", "india", 'three'),
                 ("china", "china", '6'),
                 ("japan", "china", 'Five')]

    df = spark.createDataFrame(data_list) \
        .toDF("source", "destination", "shipments")

    n_nulls = spark.sparkContext.accumulator(0)
    spark.udf.register("udf_count_nulls", count_nulls, IntegerType())
    df.withColumn("shipments_int", expr("udf_count_nulls(shipments)")) \
        .show()

    print("Bad Record Count:" + str(n_nulls.value))

```

It is always recommended to use an accumulator from inside an action, instead of from inside a transformation. Because Spark guarantees accurate results inside an action, as Spark runs duplicate tasks in many situations. Some spark tasks can fail for a variety of reasons, but the driver will retry those tasks on a different worker, assume success in the retry; it might also trigger a duplicate task if a task is running very slow. 

Spark in Scala also allows you to give your accumulator a name and show them in the Spark UI. However, PySpark accumulators are always unnamed, and they do not show up in the Spark UI. Spark allows you to create Long and Float accumulators. However, you can also create custom accumulators.

## Speculative Execution


## Dynamic Resource Allocation


## Spark Schedulers


## Unit Testing in Spark
skipped. 


