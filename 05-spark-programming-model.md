# Spark programming model
## Creating Spark Project Build Configuration
Your Spark Home should point to the correct version of Spark binaries. 

PyCharm -> Create New Project -> Give a name of your project; New environment using Conda; Python version: 3.7 -> Create

Install necessary packages. File -> Settings -> Project: (projectName) -> Project Interpreter -> + -> 
pySpark, Specify version: 2.4.5 -> Install Package -> + -> PyTest -> Install Package. 

Create the main program. In the left pane, right click on the project name folder -> New -> Python File -> Name: HelloSpark

"HelloSpark.py": 
```py
# you need this line in every spark code
from pyspark.sql import *

if __name__ == "__main__":
    print("Starting Hello Spark. ")
```

Run the code, and see the printed message. 

## Configuring Spark Project Application Logs


## Creating Spark Session


## Configuring Spark Session


## Data Frame Introduction


## Data Frame Partitions and Executors


## Spark Transformations and Actions


## Spark Jobs Stages and Task


## Understanding your Execution Plan


## Unit Testing Spark Application








