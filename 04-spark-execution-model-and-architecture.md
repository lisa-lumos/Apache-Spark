# Spark execution model and architecture
## Execution Methods - How to Run Spark Programs?
2 methods to run spark programs:
1. Interactive clients. spark-shell, notebooks, etc. Mostly for learning/exploration. 
2. Submit a job. spark-submit, Databricks Notebook, Rest API. Such as the below use cases. 

Use cases:
- Stream processing. Read news feed as a continuous stream, then apply ML to figure out the type of uses that might be interested in each news, and direct them to these users
- Batch processing. YouTube statistics, collect the data for a day, and start a scheduled spark job, to compte the watch time in minutes. The result goes into a table and a dashboard. 

For these use cases, you must package your app, and submit it to the Spark cluster for execution. 

## Spark Distributed Processing Model - How your program runs?
When an application is submitted to Spark, it will create a master process for the app. This master process will then create a bunch of workers to distribute and complete the job. 

In Spark terminology, the master is a driver, and the workers are executors.

The Spark engine asks for a container from the underlying cluster manager, to start the driver process. Once started, the driver again will ask for some more containers, to start the executor process. This happens for each application. 

## Spark Execution Modes and Cluster Managers
Q: How does Spark run on a local machine?
A: Local cluster manager, spark runs locally as a multi-threaded application. If you only use one thread, the the driver has to do all the work by itself. If you use 3 threads, you will get 1 dedicated driver thread, and 2 executor threads. Basically, it is a simulation of distributed client-server architecture, using multiple threads. 

Q: How does Spark run with an interactive tool? 
A: The client mode. The Spark driver on the local machine is connected to the cluster manager (yarn), and starts all the executors on the cluster. If you quit your client, or log off from your client machine, then your driver dies, and hence the executors will also die, in the absence of the driver. So client mode is suitable for interactive work, but not for long-running jobs. 

The cluster mode, in contrast, is designed to submit your application to the cluster, and let it run. In this mode, everything, including driver and executors, run on the cluster. Once you submit your job, you can log off from the client machine, and your driver is not impacted. 

## Summarizing Spark Execution Models - When to use What?
Cluster managers: local[n] (for using on your local machine), or YARN (for using on a real cluster). 

Execution modes: client (for local machine, or on YARN cluster with notebook/sparkShell), or cluster

Execution Tools: IDE, Notebook (for local machine), Spark submit (real cluster)

## Working with PySpark Shell - Demo
```sh
pyspark --help

# use 3 threads, give driver 2gb of ram. 
pyspark --master local[3] --driver-memory 2G

>>> df = spark.read.json("C:/demo/notebook/data/people.json")
>>> df.show()

# This url is the Spark Context web UI. It is only available when the spark application is running. 
# localhost:4040/jobs/ 

```

## Installing Multi-Node Spark Cluster - Demo
In you GCP account, go to your Console Home page. Click the dropdown menu in the upper left corner -> Dataproc -> Clusters. This ia an on-demand YARN Cluster, which comes with the Spark setup. Name: yarn-cluster; Master node Machine type: n1-standard-2 (2 vCPU, 7.5GB memory); Primary disk size: 32; Worker nodes Machine type: n1-standard-1 (1 vCPU, 3.75GB memory); Primary disk size: 32; Nodes: 3; Check the Enable access to the web interfaces -> Advance options -> Cloud Storage staging bucket: (create a bucket, in the same region as the cluster); Image: 1.5 (get the latest version of Spark); Optional components: Anaconda, Zeppelin Notebook; Scheduled deletion: check Delete after a cluster idle time period without submitted jobs: 1 hrs -> Create. 

## Working with Notebooks in Cluster - Demo
Mostly used by data scientists/analysts for interactive exploration directly in the production cluster. 

Work with shell. ssh to the GCP master node, and run:
```sh
pyspark --master yarn --driver-memory 1G --executor-memory 500M --num-executors 2 --executor-cores 1
```

In the GCP cluster page, go the WEB INTERFACES pane, and can access the Spark Context UI from here. YARN ResourceManager (show apps that are currently running) and Spark History Server (shows list of applications that you executed in the past. Clicking the application id will take you to the Spark Context UI, where you can see when drivers and executors are added. ). 

Work with Zeppelin Notebook UI. In the GCP cluster page, go the WEB INTERFACES pane -> Zeppelin
```py
# cell
# default is scala
spark.version

# cell
# need to add an interpreter directive for the cell,
# to ensure the cell is a pyspark cell
%pyspark
1 + 1 # print something
```

## Working with Spark Submit - Demo
Mostly used to executing packaged Spark application on your prod cluster. 
```sh
spark-submit --help

# the 100 is the input into the code
spark-submit --master yarn --deploy-mode cluster test.py 100
```
