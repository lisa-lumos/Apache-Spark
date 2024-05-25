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
```conf
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
The SparkSession object is your Spark Driver, which starts the executors, who then will do most of the work. 

When you start the spark shell, or the notebook, they create a Spark Session for you, and make it available as a variable, named "spark". 

However, when you are writing a Spark program, then you must create a SparkSession, because it is not already provided. 

The SparkSession is a Singleton object, so each Spark application can have one and only one active spark session. It is because SparkSession if your driver, and you cannot have more than 1 driver in a Spark application. 

"HelloSpark.py": 
```py
# you need this line in every spark code
from pyspark.sql import *

if __name__ == "__main__":
    print("Starting Hello Spark. ")

    # create a spark session
    # the builder method returns a Builder object
    # Some configs, such as appName() and master() are most commonly used,
    # so we have direct methods to specify them. 
    # But we also have some overloaded config() methods, 
    # for all other configurations. 
    # master() is the cluster manager, such as local JVM with 3 threads
    spark = SparkSession.builder \ 
        .appName("Hello Spark") \
        .master("local[3]")
        .getOrCreate()

    logger = Log4J(spark)

    logger.info("Starting HelloSpark")

    # you processing code goes here

    logger.info("Finished HelloSpark")

    spark.stop() # after done data processing, stop the driver
```

Right click on project name -> New -> Python Package -> name it as "lib" (this will create a "lib" folder under project folder, and put a "__init__.py" file in it) -> Right click on the "lib" folder -> New -> Python File -> name it as "logger" (this will create a "logger.py" file).

"logger.py":
```py
class Log4J:
    def __init__(self, spark):
        log4j = spark._jvm.org.apache.log4j
        # the logger name is defined in the "log4j.properties" file
        # use your organization name as the root class,
        # and suffix it with the application name
        # as long as root_class matches, log4j will work
        root_class = "guru.learningjournal.spark.examples"
        conf = spark.sparkContext.getConf()
        app_name = conf.get("spark.app.name")
        self.logger = log4j.LogManager.getLogger(root_class + "." + app_name)

    def warn(self, message):
        self.logger.warn(message)

    def info(self, message):
        self.logger.info(message)

    def error(self, message):
        self.logger.error(message)

    def debug(self, message):
        self.logger.debug(message)

```

Run "HelloSpark.py", and see the prints. See the newly created "app-logs" folder under the project folder. See the "hello-spark.log" file in there. Notice the logs in this file doesn't have noisy entries from other spark and hadoop packages.  

## Configuring Spark Session
For developers, the common ways to configure the spark session are:
1. spark-submit command line options. Such as `spark-submit --master local[3] --conf "spark.app.name=Hello Spark" --conf spark.eventLog.enabled=false HelloSpark.py`. Note double quotes are there because of the space. 
2. SparkConf object. Such as `.appName("...").master("...")`

"HelloSpark.py": 
```py
from pyspark.sql import *
from lib.logger import Log4J

if __name__ == "__main__":
    # a list of all spark configs: 
    # https://spark.apache.org/docs/latest/configuration.html#application-properties
    conf = SparkConf()
    conf.set("spark.app.name", "Hello Spark")
    conf.set("spark.master", "local[3]")
    
    spark = SparkSession.builder \ 
        .config(conf=conf) \
        .getOrCreate()

    logger = Log4J(spark)

    logger.info("Starting HelloSpark")

    # you processing code goes here

    logger.info("Finished HelloSpark")

    spark.stop()
```

Order of precedence for config locations: env variable < "spark-defaults.conf" file < command line options < SparkConf. So configs in the application code has the highest precedence. 

Which config method to use:
1. For deployment related configs, such as spark.driver.memory, spark.executor.instances, which depend on the cluster manager and your deploy mode. Set them through the spark-submit command line. 
2. For configs that control the Spark Application runtime behavior, such as spark.task.maxFailures. Set them through SparkConf in the code. 

In the current version of the code, the master is hard-coded. This can be a problem when you deploy to prod. 3 ways to handle it. 

Solution: Could create a config file and load them at runtime. Right-click the project folder -> New -> File -> name it "spark.conf":
```conf
[SPARK_APP_CONFIGS]
spark.app.name = Hello Spark
spar.master = local[3]
```

Right click the "lib" folder -> New -> Python file -> name it "utils":
```py
import configparser
from pyspark import SparkConf

def get_spark_app_config():
    spark_conf = SparkConf()
    config = configparser.ConfigParser()
    config.read("spark.conf")

    for (key, val) in config.items("SPARK_APP_CONFIGS"):
        spark_conf.set(key, val)
    return spark_conf
```

"HelloSpark.py": 
```py
from pyspark.sql import *
from lib.logger import Log4J
from lib.utils import get_spark_app_config

if __name__ == "__main__":
    conf = get_spark_app_config()
    spark = SparkSession.builder \ 
        .config(conf=conf) \
        .getOrCreate()

    logger = Log4J(spark)

    logger.info("Starting HelloSpark")

    # print out the config
    conf_out = spark.sparkContext.getConf()
    logger.info(conf_out.toDebugString())

    logger.info("Finished HelloSpark")

    spark.stop()
```

## Data Frame Introduction
Read -> process -> write

Assume in the project directly, there is a "data" folder, with a file "sample.csv" inside.

First, pass the file name and location as a command line argument to the application. Because we do not want to hard-code the file dir. In PyCharm, Run -> Edit Configurations... -> Under the "Configuration" pane, Parameters: data\sample.csv -> OK. 

Spark DataFrame is a 2D table-like, distributed data structure, that is inspired by Pandas dataframe. 

"lib/utils.py", add a function to read the data, so later can used for unit testing also:
```py
import configparser
from pyspark import SparkConf

def get_spark_app_config():
    spark_conf = SparkConf()
    config = configparser.ConfigParser()
    config.read("spark.conf")

    for (key, val) in config.items("SPARK_APP_CONFIGS"):
        spark_conf.set(key, val)
    return spark_conf

def load_survey_df(spark, data_file):
    # to see a list of csv read options:
    # go to Spark Python API docs
    # search for "DataFrameReader",
    # find pyspark.sql.DataFrameReader -> csv method
    # header is true, so df get inferred col names
    # inferSchema sometimes work, but not always
    return spark.read \
        .option("header", "true") \
        .option("inferSchema", "true") \
        .csv(sys.argv[1]) # pass the file path

```

"HelloSpark.py": 
```py
from pyspark.sql import *
from lib.logger import Log4J
from lib.utils import get_spark_app_config

if __name__ == "__main__":
    conf = get_spark_app_config()
    spark = SparkSession.builder \ 
        .config(conf=conf) \
        .getOrCreate()

    logger = Log4J(spark)

    # check the command line argument
    if len(sys.argv) != 2:
        logger.error("Usage: HelloSpark <filename>")
        sys.exit(-1)

    logger.info("Starting HelloSpark")

    survey_df = load_survey_df(spark, sys.argv[1])
    survey_df.show() # run it to see the results

    logger.info("Finished HelloSpark")

    spark.stop()
```

## Data Frame Partitions and Executors
The data in the HDFS is stored in a distributed storage. The data is stored in many partitions, and each node store one or more of these partitions. When spark read the data, it creates a logical in-memory data structure (a dataframe). The driver will then assign to each executor their share of partitions to process the data. While assigning partitions to these executors, Spark will try to allocate the partitions which are closest to the executors in the network, for utilize locality. 

## Spark Transformations and Actions
Spark data processing is about creating a DAG of operations. 

Narrow dependency transformation: A transformation that can be performed independently on a single partition, to produce valid results. Such as "where()". 

Wide dependency transformation: A transformation that requires data from other partitions to produce valid results. Such as "groupBy()", "orderBy()", "join", "distinct", etc. Re-partition can be used to make sure all records of the same group are collected into the same partition. This "re-partition" is known as "shuffle and sort" operation, and is managed internally by Spark.  

Lazy Evaluations. When the DAG of operations goes to the Spark driver, it will re-arrange them to optimize certain activities, and create an execution plan for the executors. The execution plan is triggered and terminated by an action (read/write/collect/show). 

Transformations are lazy, but actions are evaluated immediately. 

"HelloSpark.py": 
```py
from pyspark.sql import *
from lib.logger import Log4J
from lib.utils import get_spark_app_config

if __name__ == "__main__":
    conf = get_spark_app_config()
    spark = SparkSession.builder \ 
        .config(conf=conf) \
        .getOrCreate()

    logger = Log4J(spark)

    if len(sys.argv) != 2:
        logger.error("Usage: HelloSpark <filename>")
        sys.exit(-1)

    logger.info("Starting HelloSpark")

    survey_df = load_survey_df(spark, sys.argv[1])
    filtered_survey_df = survey_df.where("Age < 40") \
        .select("Age", "Gender", "Country") \
        .groupBy("Country")
    survey_df.show() 

    logger.info("Finished HelloSpark")

    spark.stop()
```




## Spark Jobs Stages and Task


## Understanding your Execution Plan


## Unit Testing Spark Application








