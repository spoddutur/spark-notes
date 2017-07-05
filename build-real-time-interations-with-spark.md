# Building Real-time interactions with Spark

#### “I want to expose my Spark applications for user interactivity via web…” 
Have u also got this thought? Then you are at the right place. Please follow along to get insights of different options on how to build the interactivity with spark.

## 1. Problem:
Apache Spark™ is a very fast and easy-to-use big-data processing engine. It came in as a very strong contender replacing MR computation engine on top of HDFS and kind of succeeded in it. But the problem that we are going to look into here is: 
#### “How do we enable interactive applications against Apache Spark?”

There are two widely adopted approaches to communicate with Spark and each of it comes with their own limitations when it comes to flexible interaction:
1. [spark-submit](#spark-submit) is a great handy script to submit spark application. Its great if you need to submit applications from command line. But, what if you need to submit applications from other applications that too if your code snippet is not bundled as jar? 
2. [spark-shell](#spark-shell) is a powerful tool to analyse data interactively. It lets user submit the code snippets and control the tasks executed in Spark cluster. However, unfortunately, it is not a consumable service that we could use with our applications.

## 1.1. Use-cases:
Following are some of the use-cases where the above two mentioned approaches to communicate with Spark fall short in providing the interaction user might want:
```markdown
1. Have the trained model loaded in SparkSession and quickly predict for user given query.
2. Monitor the data crunching that spark-streaming is handling live
3. Access your big-data cached in spark-cluster from outside world
4. Spawn the same Spark job iteratively as and when needed with varied parameters from UI
5. How to spawn your spark-job interactively from a web-application
6. Spark-as-a-Service via REST 
```
## 2. Solution:
There are many ways one might think of interacting with Spark while solving above use-cases. I have made following four solutions to address the discussed usecases. Hopefully it  provides some insight into how to go about building an interactive spark application that caters to your needs:

### 2.1 Cloud-based-sql-engine-using-spark: 
```markdown
Access your big-data cached in spark-cluster from outside world
```
- [<img src="https://user-images.githubusercontent.com/22542670/27858799-3b567bb4-6194-11e7-8b28-1280e9e0c1d1.png" alt="Git" width="100"/>](https://github.com/spoddutur/cloud-based-sql-engine-using-spark): [https://github.com/spoddutur/cloud-based-sql-engine-using-spark](https://github.com/spoddutur/cloud-based-sql-engine-using-spark) 
- **Objective:** Use SPARK as Cloud-based SQL Engine and expose your big-data as a JDBC/ODBC data source via the Spark thrift server.[<img src="https://user-images.githubusercontent.com/22542670/27858799-3b567bb4-6194-11e7-8b28-1280e9e0c1d1.png" alt="View on Git" width="150"/>](https://github.com/spoddutur/cloud-based-sql-engine-using-spark)
- **Architecture:**
<img src="https://user-images.githubusercontent.com/22542670/27733176-54b684c2-5db2-11e7-946b-5b5ef5595e43.png" width="600" />

### 2.2 Sparkstreaming-monitoring-with-lightning: 
```markdown
Monitor the data crunching that spark-streaming is handling live
```
- **Git:** [https://github.com/spoddutur/spark-streaming-monitoring-with-lightning](https://github.com/spoddutur/spark-streaming-monitoring-with-lightning)
- **Objective:** Monitor spark application not in terms of its health (like ganglia), but in terms of the data-crunching it is doing currently. This project will demo how to have a realtime graph monitoring system using Lightning-viz where we can plot and monitor any custom parameters of the live data that spark streaming application is processing right now.
- **Architecture:**
<img src="https://user-images.githubusercontent.com/22542670/27772206-f161509e-5f7a-11e7-907c-9d9b971cabe1.png" width="600" />

### 2.3 Spark-jetty-server: 
```markdown
How to spawn your spark-job interactively from a web-application
```
- **Git:** [https://github.com/spoddutur/spark-jetty-server](https://github.com/spoddutur/spark-jetty-server)
- **Objective:** Embed SparkContext within a Jetty web server. Git project is an end product of providing a plug-and-play maven project that spawns a spark-job from a web application.
- **Architecture:**
<img src="https://user-images.githubusercontent.com/22542670/27729358-3131ade2-5da3-11e7-8bc0-5ff0d6ec4fa5.png" />

### 2.4 spark-as-service-using-embedded-server:
```markdown
Build a REST api service on top of your ApacheSpark application
```
- **Git:** [https://github.com/spoddutur/spark-as-service-using-embedded-server](https://github.com/spoddutur/spark-as-service-using-embedded-server)
- **Objective:** The core of the application is not primarily a web-application OR browser-interaction but to have REST service performing big-data cluster-computation on ApacheSpark.
- **Architecture:**
<img src="https://user-images.githubusercontent.com/22542670/27823530-0b770dc8-60c7-11e7-9b22-c304fe3327fb.png" width="700"/>
