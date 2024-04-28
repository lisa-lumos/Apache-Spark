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








## Data lake










## Apache Spark and Databricks Cloud






















