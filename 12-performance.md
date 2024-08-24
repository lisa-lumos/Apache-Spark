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


## Repartition and Coalesce


## Dataframe Hints


## Broadcast Variables


## Accumulators


## Speculative Execution


## Dynamic Resource Allocation


## Spark Schedulers


## Unit Testing in Spark



