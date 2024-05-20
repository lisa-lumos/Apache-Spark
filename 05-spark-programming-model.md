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
Add a Log4J property file to the project folder. "log4j.properties":
```py
# Set everything to be logged to the console
# WARN is the log level, and console is the appender list
# so at the top most level, only see warnings and errors, at the console. 
log4j.rootCategory=WARN, console

# define console appender
log4j.appender.console=org.apache.log4j.ConsoleAppender
log4j.appender.console.target=System.out
log4j.appender.console.layout=org.apache.log4j.PatternLayout
log4j.appender.console.layout.ConversionPattern=%d{yy/MM/dd HH:mm:ss} %p %c{1}: %m%n

# application log
# 2nd log level. This level is named as: .guru.learningjournal.spark.examples
# This log will have INFO, and logs to console and file appenders
log4j.logger.guru.learningjournal.spark.examples=INFO, console, file
log4j.additivity.guru.learningjournal.spark.examples=false

# define rolling file appender
# the var spark.yarn.app.container.log.dir is configured by cluster admin
log4j.appender.file=org.apache.log4j.RollingFileAppender
log4j.appender.file.File=${spark.yarn.app.container.log.dir}/${logfile.name}.log
# log4j.appender.file.File=app-logs/hello-spark.log
# define following in Java System
# -Dlog4j.configuration=file:log4j.properties
# -Dlogfile.name=hello-spark
# -Dspark.yarn.app.container.log.dir=app-logs
log4j.appender.file.ImmediateFlush=true
log4j.appender.file.Append=false
log4j.appender.file.MaxFileSize=500MB
log4j.appender.file.MaxBackupIndex=2
log4j.appender.file.layout=org.apache.log4j.PatternLayout
log4j.appender.file.layout.conversionPattern=%d{yy/MM/dd HH:mm:ss} %p %c{1}: %m%n


# Recommendations from Spark template
log4j.logger.org.apache.spark.repl.Main=WARN
log4j.logger.org.spark_project.jetty=WARN
log4j.logger.org.spark_project.jetty.util.component.AbstractLifeCycle=ERROR
log4j.logger.org.apache.spark.repl.SparkIMain$exprTyper=INFO
log4j.logger.org.apache.spark.repl.SparkILoop$SparkILoopInterpreter=INFO
log4j.logger.org.apache.parquet=ERROR
log4j.logger.parquet=ERROR
log4j.logger.org.apache.hadoop.hive.metastore.RetryingHMSHandler=FATAL
log4j.logger.org.apache.hadoop.hive.ql.exec.FunctionRegistry=ERROR
``` 

The logger is the set of APIs that will be used in the application. These configs are loaded by the loggers at runtime. Appenders are the output destinations, such as console, and a log file. 

In your local machine, use file "SPARK_HOME/conf/spark-defaults.conf" to set JVM parameters, where "SPARK_HOME" is a environment variable. The last line of this file looks like: `spark.driver.extraJavaOptions -Dlog4j.configureation=file:log4j.properties -Dspark.yarn.app.containter.log.dir=app-logs -Dlogfile.name=hello-spark`. This indicates the log4j config file is the "log4j.properties" file in the project root directory. Similar to the log file dir location "app-logs", and the log file name "hello-spark". 

We cannot used python logger, because collecting python log files across multiple nodes ia not integrated with the spark. 

## Creating Spark Session


## Configuring Spark Session


## Data Frame Introduction


## Data Frame Partitions and Executors


## Spark Transformations and Actions


## Spark Jobs Stages and Task


## Understanding your Execution Plan


## Unit Testing Spark Application








