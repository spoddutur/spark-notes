# Spark as a successful contender to MapReduce
Its not uncommon for a beginner to think Spark as a replacement to Hadoop. The term "Hadoop" is interchangeably used to refer to either `Hadoop ecosystem (or) Hadoop MapReduce (or) Hadoop HDFS`. Apache Spark came in as a very strong contender to replace `Hadoop MapReduce` computation engine. 

<img width="411" alt="MapReduceVsSpark" src="https://user-images.githubusercontent.com/22542670/30010978-a1d0d456-9151-11e7-939a-8ed383cffab1.png">

This blog is to better understand what motivated Spark and how it evolved successfully as a strong contender to MapReduce.

We will section this blog in 3 parts:
1. **MapReduce computation Engine in a Nutshell**

2. **Cons of MapReduce as motivation for Spark**
    - Look at the drawbacks of MapReduce
    - How Spark addressed them
3. **How Spark works**
   - Behind the scenes of a spark application running in cluster 
4. **Appendix**
   - Look at other attempts like Corona done to make up for the downsides of MapReduce Engine.

## 1. MapReduce (MR) computation in a nutshell
I’ll not go deep into the details, but, lets see birds eye view of how Hadoop MapReduce works. Below figure shows a typical Hadoop Cluster running two Map-Reduce applications. Each of these application’s Map(M) and Reduce(R) jobs are marked with black and white colours respectively.

<img width="394" alt="Hadoop MapReduce" src="https://user-images.githubusercontent.com/22542670/30012789-389587e8-9160-11e7-8dcd-6d48e88dd085.png">

- **NameNode and DataNode:** 
    - NameNode + DataNodes essentially make up HDFS.
    - NameNode only stores the metadata of HDFS i.e., it stores the list of all files in the file system (not its data), and keeps a track of them across the cluster.
    - DataNodes store the actual data of files.
    - NameNode JVM heart beats with DataNode JVM’s every 3secs.

- **JobTracker (JT) and TaskTracker (TT):** 
    - JobTracker JVM is the brain of the MapReduce Engine and runs on NameNode.
    - JobTracker creates and allocates jobs to TaskTracker which runs on DataNodes.
    - TaskTrackers runs the task and reports task status to JobTracker.
    - __Inside TaskTracker JVM, we have slots where we run our jobs. These slots are hardcoded to be either Map slot or Reduce slot. One cannot run a reduce job on a map slot and vice-versa.__
    - __Parallelism in MapReduce is achieved by having multiple parallel map & reduce jobs running as processes in respective TaskTracker JVM slots.__

- **Job execution:** In a typical MapReduce application, we chain multiple jobs of map and reduce together.  It starts execution by reading a chunk of data from HDFS, run one-phase of map-reduce computation, write results back to HDFS,  read those results into another map-reduce and write it back to HDFS again. There is usually like a loop going on there where we run this process over and over again

## 2. Cons of Map-Reduce as motivation for Spark

One can say that Spark has taken direct motivation from the downsides of MapReduce computation system. Let’s see the drawbacks of MapReduce computation engine and how Spark addressed them:

1. **Parallelism via processes:** 
    - *MapReduce:* MapReduce doesn’t run Map and Reduce jobs as threads. They are processes which are heavyweight compared to threads.
    - *Spark:* Spark runs its jobs by spawning different threads running inside the executor.
2. **CPU Utilization:** 
    - *MapReduce:* The slots within TaskTracker, where Map and Reduce jobs gets executed, are not generic slots that can be used to run either Map or Reduce job. These slots are categorized into two types, one to run Map jobs and the other to run Reduce jobs. How does it matter? So, when you start a MapReduce application, initially, that application might spend like hours in just the Map phase. So, during this time none of the reduce slots are going to be used. This is why if you notice, your CPU% would not be high because all these Reduce slots are sitting empty. Facebook came up with an improvement to address this a bit. If you are interested please check Appendix section below.
    - *Spark:* Similar to TaskTracker in MapReduce, Spark has Executor JVM’s on each machine. But, unlike hardcoded Map and Reduce slots in TaskTracker, these slots are generic where any task can run.
3. **Extensive Reads and writes:** 
    - *MapReduce:* There is a whole lot of intermediate results which are written to HDFS and then read back by the next job from HDFS. Data handshake between the any two jobs chained together happens via reads and writes to HDFS.
    - *Spark:* Spark is an in-memory processing engine. All of the data and intermediate results are kept in-memory. This is one of the reasons that you get 10-100x faster speed because of the efficient memory leverage.

#### Note: Facebook came up with Corona to address some of these cons and it did achieve 17% performance improvements on MapReduce Jobs. I've detailed it in Appendix.

## 3. How Spark works: 
Now that we have seen the disadvantages with MapReduce and how Spark addressed it, its time to jump in and look at the internals of Spark briefly. In that, i'll mainly try to cover how a spark application running in a cluster looks like. Below picture depicts Spark cluster:

<img width="540" alt="spark-standalone-mode" src="https://user-images.githubusercontent.com/22542670/30005242-a22a0c5c-90fb-11e7-9d80-97efe540417f.png">

Let's look at differenct components shown in the above picture:
- **Spark Master, Worker and Executor JVM’s:**
SparkMaster and Worker JVM’s are the resource managers. All worker JVM’s will register themselves with SparkMaster. They are very small. Example: Master use like 500MB of RAM & Worker uses like 1GB RAM.
- **Master:**
Master JVM’s job is to decide and schedule the launch of Executor JVM’s across the nodes. But, Mater will not launch executors. 
- **Worker:**
It is the Worker who heartbeats with Master and launches the executors as per schedule. 
- **Executor:**
Executor JVM has generic slots where tasks run as threads. Also, all the data needed to run a task is cached within Executor memory.
- **Driver:**
When we start our spark application with spark submit command, a driver will start and that driver will contact spark master to launch executors and run the tasks. Basically, Driver is a representative of our application and does all the communication with Spark.
- **Task:**
Task is the smallest unit of execution which works on a partition of our data. Spark actually calls them cores. —executor-cores setting defines number of tasks that run within the Executor. For example, if we have set —executor-cores to six, then we can run six simultaneous threads within the executor JVM.
- **Resilience:**
Worker JVM’s work is only to launch Executor JVM’s whenever Master tells them to do so. If Executor crashes, Worker will restart it. If Worker JVM crashes, Master will start it. Master will take care of driver JVM restart as well. But then, if a driver restarts, all the Ex’s will have to restart. 
- **Flexible Distribution of CPU resources:**
By CPU resources, We are referring to the tasks/threads running within an executor. Let’s assume that the second machine in the cluster has lot more ram and cpu resources. Can we run more threads in this second machine? Yes! You can do that by tweaking spark-env.sh file and set SPARK_WORKER_CORES to 10 in the second machine. The same setting if set to 6 in other machines, then master will launch 10 threads/tasks in that second machine and 6 in the remaining one’s. But, you could still oversubscribe in general. SPARK_WORKER_CORES tells worker JVM as to how many cores/tasks it can give out to its underlying executor JVM’s.

## 3. Conclusion: 
We've seen:
- The initial motivation behind Spark
- Why it evolved successfully as a strong contender to MapReduce
- Why is Spark orders of magnitude faster than traditional Hadoop’s map-reduce system
- An overview of Spark application running in cluster

[My HomePage](https://spoddutur.github.io/spark-notes/)

## 4. Appendix:
### 4.1 Corona - An attempt to make up for the downsides of MapReduce and improve CPU Utilization
Facebook came up with Corona to address the CPU Utilization problem that MapReduce has. In their hadoop cluster, when Facebook was running 100’s of (MapReduce) MR jobs with lots of them already in the backlog waiting to be run because all the MR slots were full with currently running MR jobs, they noticed that their CPU utilisation was pretty low (~60%). It was weird because they thought that all the Map (M) & Reduce (R) slots were full and they had a whole lot of backlog waiting out there for a free slot. What they noticed was that in traditional MR, once a Map Job finishes, then TaskTracker has to let JobTracker know that there is an empty slot. JobTracker will then allot this empty slot to the next job. This handshake between TaskTracker & JobTracker is taking ~15-20secs before the next job takes up that freed up slot. This is because, heartbeat of JobTracker is 3secs. So, it checks with TaskTracker for free slots once in every 3secs and it is not necessary that the next job will be assigned in the very next heartbeat. So, FaceBook added Corona which is a more aggressive job scheduler added on top of JobTracker. MapReduce took 66secs to fill a slot while Corona took like 55 secs (~17%). Slots here are M or R process id’s. 

### 4.2. Legend: 
- MR - MapReduce
- M - Map Job
- R - Reduce Job
- JT - JobTracker
- TT - TaskTracker

## 5. References:
- [Spark Summit East Advanced Devops Student Slides](https://www.slideshare.net/databricks/spark-summit-east-2015-advdevopsstudentslides)
- [Apache Spark Training](https://www.youtube.com/watch?v=7ooZ4S7Ay6Y)
