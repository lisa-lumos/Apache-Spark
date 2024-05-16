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


## Spark Execution Modes and Cluster Managers


## Summarizing Spark Execution Models - When to use What?


## Working with PySpark Shell - Demo


## Installing Multi-Node Spark Cluster - Demo


## Working with Notebooks in Cluster - Demo


## Working with Spark Submit - Demo