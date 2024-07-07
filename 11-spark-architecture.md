# 11. Spark Architecture
## Apache Spark Runtime Architecture
Spark is a distributed computing platform. Spark application is a distributed application, so it needs a cluster. Hadoop YARN cluster and Kubernetes cluster covers 90%+ of spark cluster market share. 

Assume you use the spark-submit command to submit your spark app to the cluster. The request will go to the YARN resource manager. The YARN RM will create one Application Master (AM) container on a worker node, and start your app's main() method in the container. A container is an isolated virtual runtime environment, which comes with some CPU and memory allocation.

Assume your app is a pyspark app. Note that Spark is written in Scala, and it runs in the Java Virtual Machine (JVM). 

Scala is a JVM language, and it always runs in the JVM. But Spark developers wanted to bring this to Python developers, so they created a Java wrapper on top of the Scala code. Then, they created a Python wrapper (known as PySpark) on top of the Java wrappers.

You have Python code in your main() method, which is designed to start a Java main() method internally. So your PySpark application will start a JVM application. Then, the PySpark wrapper will call the Java wrapper, using the Py4J connection (which allows a Python app to call a Java app), and the Java wrapper runs Scala code in the JVM. 

The PySpark main method is the PySpark Driver, and the JVM application is the Application Driver. (for the driver, not the executor)

But if your code is written in Scala instead of PySpark, you will only have an Application Driver. 

The driver start first, then it will request the cluster RM for more containers, and start executors in these new containers (they run JVM). Note that the driver and executor can use the CPU and memory given to the container, they cannot use CPU or Memory from the workers. 

If you use some additional Python libraries, that are not part of PySpark, even if you are creating UDFs in Python, then on each executor container, besides the JVM, you also have a Python worker. 

## Spark Submit, and important options
There are many ways to submit a Spark application to your test/prod cluster. The most commonly used way is the spark-submit command-line tool. 

The general structure of the spark-submit command:
```
spark-submit --class <main-class> --master <master-url> --deploy-mode <deploy-mode> <application-jar> [application-args]
```
A list of the most commonly used options:
- `--class`: Not applicable for PySpark, only for Java/Scala
- `--master`: yarn, local[3]
- `--deploy-mode`: client or cluster
- `--conf`: spark.executor.memoryOverhead=0.20
- `--driver-cores`: 2
- `--driver-memory`: 8G
- `--num-executors`: 4
- `--executor-cores`: 4
- `--executor-memory`: 16G

Note the driver/executor cores and ram setting are for driver/executor containers. 

For example: `spark-submit --master yarn --deploy-mode cluster --driver-cores 2 --driver-memory 8G --num-executors 4 --executor-cores 4 --executor-memory 16G hello-spark.py`. 

## Deploy modes - Client and Cluster mode
In the cluster mode, the spark-submit will reach the YARN resource manager, requesting it to start the driver in the AM (application master) container. So it will start the AM container on a worker node in the cluster. Then this driver will request YARN to start some executor containers, and hand them over to the driver. 

In the client mode, the spark-submit doesn't go to the YARN resource manager for starting an AM container. Instead, the spark-submit command will start the driver JVM directly on the client machine (your local machine). Then, the driver will reach out to the YARN resource manager, requesting executor containers, the YARN RM will start the executor containers and handover them to the driver. The driver then will start executors in those containers to do the work. 

Basically, in cluster mode, your driver runs in the cluster, but in client mode, your driver runs in your client machine. Everything else are the same. 

You will almost always submit your application in cluster mode:
- It allows you to submit the application, and log off from the client machine. 
- Your app runs faster, because your driver is closer to the executors. 

The client mode is designed for interactive workloads. e.g., spark-shell runs your code in client mode. Spark notebooks also use the client mode. When a driver is local, it can easily communicate with you, get the results and show it back to you. Also, if you log off from the client machine, or stop the shell and quit, the driver dies, the YARN RM knows that since the driver is dead, the executors assigned to the driver are now orphans, so the RM will terminate the executors. 

## Spark Jobs - Stage, Shuffle, Task, Slots
Transformations are used to process/convert data, and they are categorized into:
- Narrow Dependency transformations. Can run in parallel on each data partition. select/filter/withColumn/drop(), etc
- Wide Dependency transformation. Require some kind of grouping, before they can be applied. groupBy/join/cube/rollup/agg(), etc

Actions trigger some work, such as writing df to disk, computing the transformations, collecting the results. read/write/collect/take/count(), etc.

All spark actions trigger one/more spark jobs. 

An example spark code does these 4 steps:
1. read a csv into a df, infer the schema (it is an action)
2. repartition the df into 2 partitions (which is a wide dependency transformation)
3. filter (narrow transformation), select(narrow transformation), group(wide transformation), then count the df (narrow transformation).
4. run df.collect().mkString("->"), write into the log. (it is an action)

This code has two code blocks (a block ends with an action):
1. block 1 contains step 1
2. block 2 contains step 2-4

Note that a large application may have many code blocks. This example only has 2. 

A typical spark application looks like a set of code blocks, and Spark will run each code block as one "spark job". Because each action creates a spark job, that contains all the transformations of its code block. 

Looking at the block #2: the driver will break the job into tasks, and assign them to the executors. 

Spark driver creates a logical query plan for each Spark job. This job's logical query plan: df -> repartition -> where -> select -> group by -> count -> df

This plan is then further break down into stages, with each stage ends with a wide dependency transformation, unless it is the part after the last wide dependency transformation, which itself is the last stage. So if you have n wide dependencies, your plan will have n+1 stages. 

So, spark cannot run these stages in parallel. Because the output of the first stage is the input of the next stage. 

So, the stages of block 2 are (runtime execution plan):
1. repartition. Contains a task.
2. filter, select, group by. Contains two tasks that runs in parallel, because there are 2 partitions now. 
3. count. Contains two tasks that runs in parallel, because there are 2 partitions. 

The output of a stage is stored in an exchange buffer (Write Exchange). The next stage starts with another exchange buffer (Read Exchange). Spark is distributed, so the Write Exchange and the Read Exchange might be on two different worker nodes. So sometimes need a copy of data partitions from the write exchange to the read exchange, which is the Shuffle/Sort operation. 

The Shuffle/Sort is an expensive operation in the Spark cluster. It requires a write exchange buffer, and a read exchange buffer. The data from the write exchange buffer is sent to the read exchange buffer over the network. 

The number of task in a stage is equal to the number of input partitions. 

The task is the most critical concept of a Spark job. It is the smallest unit of work in a Spark job. The Spark driver assigns tasks to the executors, and asks them to do the work. 

The executor needs the task code, and data partition, to perform the work. 

Assume your Spark cluster has a driver, and 4 executors, with each executor has one JVM process. Assume you assigned 4 CPU cores to each executor, so each executor JVM can create 4 parallel threads (aka, the slot capacity of the executor, the 4 executor slots on each executor). 

The driver knows how many slots each executor has, and it will assign tasks to fit in the executor slots. Assume a stage has 10 input data partitions, so it has 10 tasks, then the driver will assign these 10 tasks to 16 slots, so there will be 6 slots sits idle, because we do not have enough tasks for all the slots. Assume you have 32 tasks for the next stage. The driver will schedule 16 tasks in the available slots, the remaining 16 will wait for slots to become available again. 

The collect() action requires each task to send data back to the driver, so the tasks of the last stage will send the result back to the driver over the network. The driver will collect data from all the tasks, and present to you. 

If you had an action to write the result in a data file, then all the tasks will write a data file partition, and send the partition details to the driver. 

The driver considers the job done when all the task are successful. If any task fails, the driver might want to retry it, so it can restart the task at a different executor. If all retries also fail, then the driver returns an exception, and marks the job as failed. 

## Spark SQL Engine and Query Planning
Apache Spark provides 2 prominent interfaces to work with data:
1. Spark SQL. Compliant with ANSI SQL2003 standard. 
2. Dataframe API, which internally uses Dataset APIs (which are not available in PySpark). Allows for functional programming. 

When you use Dataframe API, spark will take action and all its preceding transformations to consider it a single job. If you write a SQL expression, Spark considers one SQL expression as one job. 

User creates the logical query plan, which then goes to the Spark SQL engine. Note that no matter you use Dataframe APIs or SQL, they all will go to the Spark SQL engine. 

Spark SQL Engine will process your logical plan in 4 stages:
1. Analysis stage. Parse your code for errors, and incorrect names, etc, and create a fully resolved logical plan. It looks up the catalog to resolve column names, data types, etc. 
2. Logical optimization. Apply the standard rule-based optimizations to the logical plan, and create an Optimized logical plan. 
3. Physical planning. Takes a logical plan, and generate one/more physical plans. Apply cost-based optimization (calculate each physical plan's cost, and select the plan with the least cost). 
4. Code generation. The best physical plan goes into code generation, and the engine generates RDD operations.  
