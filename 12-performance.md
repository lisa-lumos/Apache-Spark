# 12. Performance
## Spark Memory Allocation
You can ask for the driver memory using below configs:
- `spark.driver.memory`. Heap memory for the spark driver JVM. Assume you asked for 1GB. 
- `spark.driver.memoryOverhead`. Used by the container process, or, any other non-JVM process within the container. Default is 10% of the 1st param, or 384MB, whichever is bigger. In this case, 1GB * 10% = 100MB < 384MB, so the value is set to 384MB.  

So, in this case, the total memory of this driver container is 1GB + 384MB. If these memory limits are violated, you will see an Out Of Memory exception. 

For each executor container, the total memory is the sum of below:
1. `spark.executor.memory`. Heap memory for JVM. Assume you asked for 8GB. 
2. `spark.executor.memoryOverhead`. Overhead memory. Default is 10% of the 1st param. Also used for network buffers, such as shuffle exchange, reading partition data from remote storage, etc. So in this case, 800MB. 
3. `spark.executor.offHeap.size`. Off heap memory. Default is 0. 
4. `spark.executor.pyspark.memory`. PySpark memory. Default is 0. 

So, in this case, the total memory of each executor is 8GB + 800MB. If these memory limitations are violated, you also will see an OOM exception.

But whether you can get a 8.8GB ram container or not, depends on your cluster config. If your worker node is a 6GB machine, you won't have enough physical memory to begin with. So you should check with your cluster admin for the maximum allowed value. If you use YARN Resource Manager, you should look for configs `yarn.scheduler.maximum-allocation-mb`, and `yarn.nodemanager.resource.memory-mb`. 

For example, if you use AWS c4.large instance type to set up your Spark cluster, using AWS EMR, the c4.large instance comes with 3.75GB RAM. When you launch the EMR cluster, it will start with the default value of `yarm.scheduler.maximum-allocation-mb = 1792`, so you cannot ask for a container higher than 1.792GB. 

PySpark is not a JVM process. So in this case, it consumes the memory allocated to the overhead. 

Both the heap memory and the overhead memory are critical for your Spark application. More than often, lack of enough overhead memory will cost you an OOM exception. 

## Spark Memory Management


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



