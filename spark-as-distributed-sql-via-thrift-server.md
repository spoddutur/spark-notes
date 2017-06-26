## Spark as a distributed backend:
Traditional relational Database engines like SQL had scalability problems and so evolved couple of SQL-on-Hadoop frameworks like Hive, Cloudier Impala, Presto etc. These frameworks are essentially cloud-based solutions and they all come with their own limitations as listed in the table below

<img width="660" src="https://user-images.githubusercontent.com/22542670/27549999-a03c529a-5abb-11e7-958b-c53f55e162f9.png">

In this blog, we’ll discuss one more such Cloud-based SQL engine using SPARK which supports both:
1. **in-memory table** - scoped to the cluster. Data is stored in Hive’s in-memory columnar format
2. **permanent, physical table** - stored in S3 using the Parquet format for data

Data from multiple sources can be pushed into Spark and then exposed as a table in one of the two mentioned approaches discussed above. Either ways, these tables are then made accessible as a JDBC/ODBC data source via the **Spark thrift server**.

### Spark Thrift Server:
Spark thrift server is pretty similar to HiveServer2 thrift. But, HiveServer2 submits the sql queries as Hive MapReduce job whereas Spark thrift server will use Spark SQL engine which underline uses full spark capabilities. 

### Example walkthrough:
Let’s walk through an example of how to use Spark as a distributed data backend engine
(code written in scala 2.11). I've used Spark 2.1.x and amazon EMR cluster with YARN for this:

1. For this example, am just loading data from a HDFS file and registering it as table with SparkSQL

```markdown
// Create SparkSession
val spark1 = SparkSession.builder()
	.appName(“SparkSql”)
	.config("hive.server2.thrift.port", “10000")
	.config("spark.sql.hive.thriftServer.singleSession", true)
	.enableHiveSupport()
	.getOrCreate()

import spark1.implicits._

// load data from /beeline/input.json in HDFS
val records = spark1.read.format(“json").load("beeline/input.json")

// As we discussed above, i'll show both the approaches to expose data with SparkSQL (Use any one of them):
// APPROACH 1: in-memory temp table:
records.createOrReplaceTempView(“records")

// APPROACH 2: parquet-format physical table in S3
spark1.sql("DROP TABLE IF EXISTS records")
ivNews.write.saveAsTable("records")
```

### We can run above code in 2 ways to register the table with Spark:
1. using spark-shell as follows:
```markdown
spark-shell --conf spark.sql.hive.thriftServer.singleSession=true
```
2. using spark-submit (bundle above code as a mvn project and create jar file out of it)
```markdown
spark-submit MainClass --master yarn-cluster <JAR_FILE_NAME>
```

### Accesing the data
1. Within the cluster
2. From a remote machine

### Accessing the data - Within the Cluster:
Perhaps the easiest way to test is connect to spark thrift server from one of the nodes in the cluster using a command line tool [beeline](https://cwiki.apache.org/confluence/display/Hive/HiveServer2+Clients#HiveServer2Clients-Beeline–NewCommandLineShell): 

```markdown
// connect to beeline
`$> beeline`
Beeline version 2.1.1-amzn-0 by Apache Hive

// within beeline, connect to spark thrift server @localhost:10000 (default host and port)
`beeline> !connect jdbc:hive2://localhost:10000`
Connecting to jdbc:hive2://localhost:10000
Enter username for jdbc:hive2://localhost:10000:
Enter password for jdbc:hive2://localhost:10000:
Connected to: Apache Hive (version 2.1.1-amzn-0)
Driver: Hive JDBC (version 2.1.1-amzn-0)
17/06/26 08:35:42 [main]: WARN jdbc.HiveConnection: Request to set autoCommit to false; Hive does not support autoCommit=false.
Transaction isolation: TRANSACTION_REPEATABLE_READ

// list all the tables and you should find our table `records` here..YeAHHHH!!!
`0: jdbc:hive2://localhost:10000> show tables;`
INFO  : OK
+-------------+
|  tab_name   |
+-------------+
| records     |
+——————+------+
```

### Accessing the data - From a remote machine:
Two things todo for this:
1. Start ThriftServer in remote with proper master url (spark://<_IP-ADDRESS_>:7077)
```markdown
$ $SPARK_HOME/sbin/start-thriftserver.sh --master spark://<_IP-ADDRESS_>:7077
starting org.apache.spark.sql.hive.thriftserver.HiveThriftServer2, logging to /Users/surthi/Downloads/spark-2.1.1-bin-hadoop2.7/logs/spark-surthi-org.apache.spark.sql.hive.thriftserver.HiveThriftServer2-1-P-Sruthi.local.out
```

2. Connect to spark thrift server using beeline (jdbc:hive2://<_IP-ADDRESS_>:10000)
```markdown
$ `$SPARK_HOME/bin/beeline`
Beeline version 1.2.1.spark2 by Apache Hive
`beeline> !connect jdbc:hive2://<_IP-ADDRESS_>:10000`
Connecting to jdbc:hive2://<_IP-ADDRESS_>:10000
`Enter username for jdbc:hive2://<_IP-ADDRESS_>:10000:<USERNAME-BY_DEFAULT_BLANK>`
`Enter password for jdbc:hive2://<_IP-ADDRESS_>:10000:<PASSWORD-BY_DEFAULT_BLANK>`
17/06/26 14:20:13 INFO jdbc.Utils: Supplied authorities: <_IP-ADDRESS_>:10000
17/06/26 14:20:13 INFO jdbc.Utils: Resolved authority: <_IP-ADDRESS_>:10000
17/06/26 14:20:13 INFO jdbc.HiveConnection: Will try to open client transport with JDBC Uri: jdbc:hive2://<_IP-ADDRESS_>:10000
Connected to: Apache Hive (version 2.1.1-amzn-0)
Driver: Hive JDBC (version 1.2.1.spark2)
Transaction isolation: TRANSACTION_REPEATABLE_READ

`0: jdbc:hive2://<_IP-ADDRESS_>:10000> show tables;`
+-------------+
|  tab_name   |
+-------------+
| records  |
+-------------+
1 row selected (1.703 seconds)

