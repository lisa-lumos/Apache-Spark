# Spark execution model and architecture
## Execution Methods - How to Run Spark Programs?
2 methods to run spark programs:
1. Interactive clients. spark-shell, notebooks, etc. Mostly for learning/exploration. 
2. Submit a job. spark-submit, Databricks Notebook, Rest API. Such as the below use cases. 

Use cases:
- Stream processing. Read news feed as a continuous stream, then apply MLto figure out the type of uses that might be interested in each news, and direct them to these users
- Batch processing. YouTube statistics, collect the data for a day, and start a scheduled spark job, to compte the watch time in minutes. The result goes into a table and a dashboard. 

For these use cases, you must package your app, and submit it to the Spark cluster for execution. 

## Spark Distributed Processing Model - How your program runs?
When an application is submitted to Spark, it will create a master process for the app. This master process will then create a bunch of workers to distribute and complete the job. 

In Spark terminology, the master is a driver, and the workers are executors.

The Spark engin ask for a container from the underlying cluster manager, to start the driver process. Once started, the driver again will ask for some more containers, to start the executor process. This happens for each application. 

## Spark Execution Modes and Cluster Managers
Q: How does Spark run on a local machine?
A: Local cluster manager, spark runs locally as a multi-threaded application. If you only use one thread, the the driver has to do all the work by itself. If you use 3 threads, you will get 1 dedicated driver thread, and 2 executor threads. Basically, it is a simulation of distributed client-server architecture, using multiple threads. 

Q: How does Spark run with an interactive tool? 
A: The client mode. The Spark driver on the local machine is connected to the cluster manager (yarn), and starts all the executors on the cluster. If you quit your client, or log off from your client machine, then your driver dies, and hence the executors will also die, in the absence of the driver. So client mode is suitable for interactive work, but not for long-running jobs. 

The cluster mode, in contrast, is designed to submit your application to the cluster, and let it run. In this mode, everything, including driver and executors, run on the cluster. Once you submit your job, you can log off from the client machine, and your driver is not impacted. 

## Summarizing Spark Execution Models - When to use What?


## Working with PySpark Shell - Demo


## Installing Multi-Node Spark Cluster - Demo


## Working with Notebooks in Cluster - Demo


## Working with Spark Submit - Demo