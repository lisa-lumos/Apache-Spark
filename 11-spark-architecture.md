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


## Spark SQL Engine and Query Planning



















