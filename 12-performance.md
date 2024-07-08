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

## Spark Adaptive Query Execution


## Spark AQE Dynamic Join Optimization


## Handling Data Skew in Spark Joins


## Spark Dynamic Partition Pruning


## Data Caching in Spark


## Repartition and Coalesce


## Dataframe Hints


## Broadcast Variables


## Accumulators


## Speculative Execution


## Dynamic Resource Allocation


## Spark Schedulers


## Unit Testing in Spark



