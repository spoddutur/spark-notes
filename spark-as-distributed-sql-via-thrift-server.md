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
Let’s walk through an example of how to use Spark as an SQL engine in YARN cluster (written in scala):
I've used Spark 2.1.0 and a EMR cluster with YARN for this. 

```markdown
val spark1 = SparkSession.builder()
	.appName(“SparkSql”)
	.config("hive.server2.thrift.port", “10000")
	.config("spark.sql.hive.thriftServer.singleSession", true)
	.enableHiveSupport()
	.getOrCreate()

import spark1.implicits._

val records = spark1.read.format(“json").load("beeline/input.json")

// approach1: in-memory temp table:
records.createOrReplaceTempView(“records")

// approach2: parquet-format physical table in S3
spark1.sql("DROP TABLE IF EXISTS records")
ivNews.write.saveAsTable("records")
```
