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

To run the code in the cell, click on the "run cell" button at the upper-right corner of the cell. 

The "Search" menu in the left pane allow you to search across folders and Notebooks. 

The "Data" menu in the left pane shows the data. You can also create tables in there. 

The "Compute" menu in the left pane shows all your clusters. After 1-2 hours of idle, the cluster will be terminated. But you can create a new one when you come back. 

The "Workflows" menu in the left pane allows you to create a workflow of jobs (aka, pipelines). But the community edition doesn't have it, without upgrading to paid version. 

In Admin Setting in the upper-right corner, you can add user, create global init scripts, etc.

In the Notebook view, in the upper-left corner, you can create new notebook, clone notebook, view setting, etc. 

## Create your First Spark Application in Databricks Cloud
In the left pane, Create -> Notebook -> Name: 01-getting-started -> Create

Assume you have data file "/databricks-datasets/ggplot2/diamonds.csv", 

Cell 1: 
```python
diamonds_df = spark.read.format("csv") \
              .option("header", "true") \
              .option("inferSchema", "true") \
              .load("/databricks-datasets/ggplot2/diamonds.csv")

diamonds_df.show(10)
```

Click the dropdown menu below the Notebook name, and create a cluster. Then click on the same dropdown menu, and attach this cluster to the notebook. 

Run Cell 1. It will display the data frame. 

Cell 2 groups the df by color, and takes avg price for each color: 
```python
from pyspark.sql.functions import avg
results_df = diamonds_df.select("color", "price") \
             .groupBy("color") \
             .agg(avg("price")) \
             .sort("color") \

results_df.show()
```

Cell 3 formats the results into tabular format:
```py
display(results_df)
```

Run cell 3, see the table, then click on the "bar chart" icon below the table, and Databricks will display it as a bar chart. 

## Setup your Local Development IDE
skipped

## Mac Users - Setup your Local Development IDE
skipped

## Create your First Spark Application using IDE
Open PyCharm, create a new project. Specify the folder of the project, use Virtualenv as new environment, select the latest interpreter, uncheck everything else -> Create. 

Right click the project folder in the left pane -> New -> Python File, and name it "HelloSpark.py". 

To check if PySpark is installed in your project venv, in the bottom bar, click "Python Packages", it will show the installed packages. If you cannot find it, search "pyspark" it in the search bar, select the correct one in the results, in the right pane, click the "three vertical dots", and "Install". 

```py
from pyspark.sql import *

if __name__ = "__main__":
    print("Hello Spark")

    spark = SparkSession.builder \
        .appName("Hello Spark") \
        .master("local[2]") \
        .getOrCreate()

    data_list = [
        ("apple", 5),
        ("orange", 10),
        ("grapes", 12)
    ]

    df = spark.createDataFrame(data_list).toDF("name", "count")
    df.show()

```

Run the code. 
