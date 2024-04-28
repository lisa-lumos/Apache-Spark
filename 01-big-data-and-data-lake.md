# Big data and Data Lake
## Big data
COBOL (Common Business-Oriented Language, designed in 1959) was a programming language designed for business data processing. It allows storing data in files, creating index files, and process data efficiently. 

Then data processing shifted from COBOL to relational databases (1970s). 

Then Hadoop emerged as a data lake platform, Cloud emerged as a platform to offer services. 

Everything else comes an goes, but only data grows. 

The big data problem: variety of data, volume of data, the velocity of new data coming in. RDBMS failed to handle this big data problem. 

Approaches of big data solution:
1. Monolithic approach. Design large/robust system. Such as Teradata and ExaData. Mostly support structured/semi-structured data. 
2. Distributed approach. Uses a cluster of computers, connected to work as a single system. 

Method 1 has vertical scalability, method 2 has horizontal scalability. The latter is easier to implement. Also, method 2 has better fault tolerance, and is more cost-effective. 

This is how Hadoop came into existence. The core platform layer offered 3 capabilities:
- Cluster operating system (YARN), an OS that makes many computers work like a single computer
- Distributed storage (HDFS), to save/read data files as if on a single machine
- Distributed computing (Map/Reduce)

On top of this Hadoop Core platform, the community developed many other tools, such as Hive database, HBase database, Pig scripting language, Sqoop  data ingestion tools, ozzie workflow tool, etc. 

Hadoop offers PB-scale of data storage. Could store structured, semi-structured, unstructured data; Hadoop offers SQL querying using Hive. Offers JDBC/ODBC connectivity. 

## Hadoop architecture, history, and evolution
### YARN
YARN is the Hadoop Cluster resource manager. It has 3 main components:
- Resource manager
- Node manager
- Application Master

Installing/configuring a Hadoop cluster is as simple as installing software on your computer. 

Hadoop uses a master-worker architecture - one of the machines will become the master, and the remaining will act as worker nodes. 

To run an application on Hadoop, you must submit the application to the YARN "resource manager". Then the "resource manager" will request one of the "node managers" to start a resource container, and run an "application master" in the container. A container is a set of resources that includes memory and CPU. 

This process will repeat if you submit another application. Each application on YARN runs inside a different "application master (AM)" container. 

### HDFS
HDFS (Hadoop distributed file system), it has 2 components:
- Name node. Installed on the master node. 
- Data node. Runs on the worker nodes. 

Assume you want to copy a large data file on your Hadoop cluster, by initiating the file copy command. This command will go to the name node. The name node will redirect this command to one/more data nodes. Assume you it re-directed to 3 data nodes. The file copy command will split the file into smaller parts (known as blocks, typically 128MB), and write those parts on the 3 data nodes. The name node keeps track of all the file metadata (file name, file directory, file size, how many blocks, sequence of the blocks). 

When you initiate a read operation, the request goes to the name node, who owns all the info for re-assembling the file from the blocks stored in the data nodes. The name node re-direct the read operation to the target data nodes. The read API will receive the data blocks from the data node, and re-assemble the file using the metadata provided by the name node. 

### Map-Reduce
It is a programming model and framework. The M/R framework is a set of APIs/services, that allow you to apply the map-reduce programming model. 

The Hadoop Map Reduce framework is now outdated, and not used anywhere. However, the Map-Reduce model is critical to understand, because it is still used in many places. 

For example, to count the number of lines in a 20TB data file, the map() function will open each block on the data node, and count the lines, then, all the data node send their counts to the reduce() function, which sum up all the counts from each data node. 

The Hive database offered by Hadoop allow you to write SQL, then the Hive SQL engine translates all your SQL expression into Map Reduce programs. Now we do not use the raw Map-reduce in Hadoop, instead, we use high-level solutions, such as Hive SQL, Spark SQL, Spark Scripting, etc. 

### The history
Google was the first company to realize the big data problem, and the first to develop a solution. They were creating a search engine, so they need to discover and crawl the web pages over the Internet, and capture the content and metadata. Then they need to store and manage it, query it. 

Google published these related details in white papers. The open-source community used these to form the basis, to develop a similar open-source implementation, which is the Hadoop. 

Apache Hive was one of the most popular components of Hadoop. It allows for creating databases/tables/views, and run SQL queries on the Hive tables. So it simplified using Hadoop. 

This Hive/Hadoop solution became very popular. However, 
- Hive SQL queries performed slower than the RDBMS SQL queries. Hadoop M/R was only available in Java, industry wanted to support other languages. 
- It used to be cheaper to add more computers for added storage, but then Cloud platforms started to offer storage at cheaper prices, making adding computers for storage expensive in comparison. So the industry wanted to use the cheaper Cloud storage. 
- Experts wanted to try other lightweight container options. Compared with the containers that YARN has to provide. 

This is when Apache Spark came into existence, as an improvement over the Hadoop Map/Reduce. It worked 10 to 100 times faster than the Hadoop M/R programs. Like Hive, Spark also offered SQL, but Spark SQL was much faster than the Hive SQL. 

Spark offered composable APIs, instead of a complex M/R model. The composable APIs were much simpler to adopt for designing data processing solutions. 

Spark supports Java, Scala, Python, and R programming languages. 

It also de-coupled the HDFS, and started supporting cloud storage, such as Amazon S3 and Azure Blob storage. 

Basically, Spark started as an improvement over Hadoop M/R, but overtime, it became an independent system. 

Hadoop was revolutionary, but now it is losing its place and importance, to the Spark platform. 

Today, Spark exists on 2 kinds of platforms:
1. Hadoop Data Lake
2. Cloud Lake house. The Databricks Spark platform is the driving force. 

## Data lake










## Apache Spark and Databricks Cloud






















