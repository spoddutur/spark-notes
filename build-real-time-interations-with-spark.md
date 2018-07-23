# Building Real-time interactions with Spark

#### “I want to expose my Spark applications for user interactivity via web…” 
Have u also got this thought? Then you are at the right place. Please follow along to get insights of different options on how to build the interactivity with spark.

## 1. Problem:
Apache Spark™ is a very fast and easy-to-use big-data processing engine. It came in as a very strong contender replacing Map-Reduce computation engine on top of HDFS and kind of succeeded in it. But the problem that we are going to look into here is: 
[How do we enable interactive applications against Apache Spark?](#prblm-stmt)

## 1.0 Limited interaction:
There are two widely adopted approaches to communicate with Spark and each of it comes with their own limitations when it comes to flexible interaction:
1. [spark-submit](https://spark.apache.org/docs/latest/submitting-applications.html) is a great handy script to submit spark application. Its great if you need to submit applications from command line. But, it doesnt support some other cases like submitting spark applications from other applications that too if your code snippet is not bundled as jar
2. [spark-shell](https://spark.apache.org/docs/latest/quick-start.html) is a powerful tool to analyse data interactively. It lets user submit the code snippets and control the tasks executed in Spark cluster. However, unfortunately, it is not a consumable service that we could use with our applications.

Please refer to Appendix section below to know some more options that spark provides to submit spark-jobs programmatically

## 1.1. Use-cases which demand flexible interaction:
Following are some of the use-cases where the above two mentioned approaches to communicate with Spark fall short in providing the interaction user might want:
```markdown
1. Have the trained model loaded in SparkSession and quickly predict for user given query.
2. Monitor the data crunching that spark-streaming is handling live
3. Access your big-data cached in spark-cluster from outside world
4. How to spawn your spark-job interactively from a web-application
5. Spark-as-a-Service via REST 
```
## 2. Solution:
There are many ways one might think of interacting with Spark while solving above use-cases. I have implemented following four solutions to address above discussed usecases and shared the corresponding github repository links where you can find further details about them. Hopefully it provides some insight into how to go about building an interactive spark application that caters to your needs:

#### Note: For each usecase listed below, I've give my git repository which not only explains the concept in detail but is also a working demo.

### 2.1 Access your big-data cached in spark-cluster from outside world:
- **Git Repository:** [https://github.com/spoddutur/cloud-based-sql-engine-using-spark](https://github.com/spoddutur/cloud-based-sql-engine-using-spark) 
- **Objective:** Use SPARK as Cloud-based SQL Engine and expose your big-data as a JDBC/ODBC data source via the Spark thrift server. 
- **Architecture:**

<img src="https://user-images.githubusercontent.com/22542670/27733176-54b684c2-5db2-11e7-946b-5b5ef5595e43.png" width="400" />

### 2.2 Monitor the data crunching that spark-streaming is handling live:
- **Git Repository:** [https://github.com/spoddutur/spark-streaming-monitoring-with-lightning](https://github.com/spoddutur/spark-streaming-monitoring-with-lightning)
- **Objective:** Monitor spark application not in terms of its health (like ganglia), but in terms of the data-crunching it is doing currently. This project will demo how to have a realtime graph monitoring system using Lightning-viz where we can plot and monitor any custom parameters of the live data that spark streaming application is processing right now.
- **Architecture:**

<img src="https://user-images.githubusercontent.com/22542670/27772206-f161509e-5f7a-11e7-907c-9d9b971cabe1.png" width="600" />

### 2.3 How to spawn your spark-job interactively from a web-application: 
- **Git Repository:** [https://github.com/spoddutur/spark-jetty-server](https://github.com/spoddutur/spark-jetty-server)
- **Objective:** Embed SparkContext within a Jetty web server. This project provides a plug-and-play maven project that spawns a spark-job from a web application.
- **Architecture:**

<img src="https://user-images.githubusercontent.com/22542670/27729358-3131ade2-5da3-11e7-8bc0-5ff0d6ec4fa5.png" />

### 2.4 Build a REST api service on top of your ApacheSpark application:
- **Git Repository:** [https://github.com/spoddutur/spark-as-service-using-embedded-server](https://github.com/spoddutur/spark-as-service-using-embedded-server)
- **Objective:** The core of the application is not primarily a web-application OR browser-interaction but to have REST service performing big-data cluster-computation on ApacheSpark.
- **Architecture:**
<img src="https://user-images.githubusercontent.com/22542670/27865894-ee70d42a-61b1-11e7-9595-02b845a9ffae.png" width="600"/>

## 3. Conclusion

#### Hopefully, this gave a better perspective on building interactive Spark applications depending on what you need. Feel free to run through my git repositories mentioned above for more in-depth consideration with a working demo for each usecase.

## 4. Appendix:
I'll list two other ways that spark provides to launch spark applications programmatically:
### 4.1 SparkLauncher
SparkLauncher is an option provided by spark to launch spark jobs programmatically as shown below. Its available in ```spark-launcher``` artifact:
```markdown
SparkAppHandle handle = new SparkLauncher()
    .setSparkHome(SPARK_HOME)
    .setJavaHome(JAVA_HOME)
    .setAppResource(pathToJARFile)
    .setMainClass(MainClassFromJarWithJob)
    .setMaster("MasterAddress
    .startApplication();
    // or: .launch().waitFor()
```
**Drawback:**
1. Spark job you want to submit should be bundled as jar. It doesnt support execution of code snippets.
2. To execute this on spark-cluster, we've to manually copy JAR file to all nodes

### 4.2 Spark REST Api
This is another alternative provided by ApacheSpark, which is similar to SparkLauncher, to submit spark jobs in a RESTful way as shown below:
```markdown
curl -X POST http://spark-cluster-ip:6066/v1/submissions/create --header "Content-Type:application/json;charset=UTF-8" --data '{
  "action" : "CreateSubmissionRequest",
  "appResource" : "file:/myfilepath/spark-job-1.0.jar",
  "clientSparkVersion" : "1.5.0",
  "mainClass" : "com.mycompany.MyJob",
  ...
}'
```
[This](http://arturmkrtchyan.com/apache-spark-hidden-rest-api) is a good reference to know about Spark REST api in detail.

**Drawbacks:**
- Similar to SparkLauncher, it only supported submissions through jars 
- It was lagging behind Apache Spark in terms of version support.

<hr/>
[My HomePage](https://spoddutur.github.io/spark-notes/)
