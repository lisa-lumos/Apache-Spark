# 2. Installing and Using Apache Spark
## Spark Development Environments
There are 2 standard ways and Python IDE to setup your Spark development environment:
1. Notebook, or Databricks Cloud, common for for Cloud platforms
2. Python IDE, or Cloudera platform, common for for On-prem platforms

Databricks Cloud has a free community edition. 

## Setup your Databricks Community Cloud Environment
In the browser, search for "try databricks" to find the free trial page. Databricks offers 14 days free trial of the Databricks Cloud product, however, you must have a cloud account. 

After filling in the information, click "Continue" and see the choose a cloud provider page. Do not choose it (it is for 15 day free trial only), instead, click on the "Get started with Community Edition". Check your inbox, click the link inside it, set your password, and confirm, to enter the Databricks Cloud Workspace home page. Log out from here.

In the browser, search for "databricks community edition", go to the login page. Bookmark this page. Login id is your email address. 

## Introduction to Databricks Workspace
In the left pane, click "Workspace". It allow you to create project directories. There are two default workspaces:
- Shared. For sharing your project code with other users. 
- Users. Contains a home directory for each user. It is recommended to create your project directories inside your home directory. 

Click on your user's home directory, in the new pane, click on the drop down icon under username -> Create -> Folder -> New Folder Name: MyTestProject -> Create Folder. 

Once this project directory (folder) is created, click on the drop down icon near it, you can clone/move/rename/delete it. Drop down icon -> Create -> Notebook.

Notebook is a code file. You can rename it after creation. In the highlighted cell, write:
```py
df = (spark.read
    .format("csv")
    .option("header", "true"))
    .option("inferSchema", "true")
    .load("/FileStore/sample_data/sample.csv")
)

display(df)
```

To execute this code, we need a cluster machine. Databricks provides a single node cluster for free, for the Community Edition. 

In the left pane, click "Compute" -> Create compute -> Compute name: My Cluster; Databricks runtime version: Standard, (latest version) -> Create compute. 

In the background, it creates a VM in the AWS Cloud. It usually takes 5 min to start a cluster. 

To connect this cluster to the previous Notebook, click "Connect" -> choose the newly-created cluster "My Cluster". 

To read a file from a location, you also need storage. You get some free storage from the Community Edition too. In the left pane, click "Data" -> DBFS (databricks file system, which is a wrapper of S3). If the DBFS is not showing up, go to the upper right corner, click on the user name -> Admin Settings -> Workspace settings -> DBFS File Browser -> Toggle it to Enabled. 

By default, there is a "FileStore" directory. Click on this folder. Upload -> DBFS Target Directory: /FileStore/sample_data; drag the csv file to Files -> Done. 











## Create your First Spark Application in Databricks Cloud



## Setup your Local Development IDE



## Mac Users - Setup your Local Development IDE



## Create your First Spark Application using IDE






















