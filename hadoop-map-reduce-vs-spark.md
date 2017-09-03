# Beginner’s myths about Spark
Its not uncommon for a beginner to think Spark as a replacement to Hadoop. This blog is to understand what is Spark and its purpose.
**Spark doesn't replace Hadoop. It is a very strong contender replacing MR computation engine on top of HDFS**
```markdown
HADOOP = HDFS + YARN + MR
Spark-HADOOP = HDFS + YARN + Spark-Core
```

Let’s try and understand how Spark is orders of magnitude faster than traditional Hadoop’s map-reduce system. For this, we will see:
```markdown
1. How Map-Reduce works
2. Disadvantages/hotspots in Map-Reduce as motivation for Spark
3. How Spark works
```

## 1. How Map-Reduce works
### 1.1 Computation in Map-Reduce system in a nutshell
I’ll not go deep into the details, but, lets see birds eye view of how Hadoop MapReduce works. Below figure shows a Hadoop cluster.
<img width="579" alt="MR-Job-Details" src="https://user-images.githubusercontent.com/22542670/30005218-08c2d5b2-90fb-11e7-97c3-532cd8fd7417.png">
- **NameNode and DataNode:** NameNode + DataNodes essentially make up HDFS. NameNode JVM heart beats with DataNode JVM’s every 3secs.
- **JobTracker and TaskTracker:** JobTracker JVM is the brain of the MapReduce Engine. TaskTracker JVM’s runs in each machine. Inside TaskTracker JVM, we have slots where we run our jobs. These slots are hardcoded to be either Map slot or Reduce slot. One cannot run a reduce job on a map slot and vice-versa.
Parallelism in MapReduce is achieved by having multiple parallel map & reduce jobs running as processes in respective TaskTracker JVM slots.
- **Job execution:** In a typical MapReduce application, we chain multiple jobs of map and reduce together.  It starts execution by reading a chunk of data from HDFS, run one-phase of map-reduce computation, write results back to HDFS,  read those results into another map-reduce and write it back to HDFS again. There is usually like a loop going on there where we run this process over and over again

## 2. Disadvantages/hotspots in Map-Reduce as motivation for Spark
- **Parallelism via processes:** 
    - *MapReduce:* MapReduce doesn’t run Map and Reduce jobs as threads. They are processes which are heavyweight compared to threads.
    - *Spark:* Spark runs its jobs by spawning different threads running inside the executor.
- **CPU Utilization:** 
    - *MapReduce:* Note that the slots within TaskTracker are not generic slots that can be used for either Map or Reduce jobs. How does it matter? So, when you start a MR application, initially, that MR app might just spend like hours in just the Map phase. So, during this time none of the reduce slots are going to be used. This is why if you notice, your CPU% would not be high because all these Reduce slots are sitting empty. Facebook came up with an improvement to address this a bit. If you are interested refer to the details in appendix.
    - *Spark:* Similar to TaskTracker in MapReduce, Spark has Executor JVM’s on each machine. But, unlike hardcoded Map and Reduce slots in TaskTracker, these slots are generic where any task can run.
- **Extensive Reads and writes:** 
    - *MapReduce:* There is a whole lot of intermediate results which are written to HDFS and then read back by the next job from HDFS. Data handshake between the any two jobs chained together happens via reads and writes to HDFS.
    - *Spark:* Spark is an in-memory processing engine. All of the data and intermediate results are kept in-memory. This is one of the reasons that you get 10-100x faster speed because of the efficient memory leverage.
As you can see, One can say that Spark has taken 1:1 motivation with the above discussed disadvantages. Let’s see the details of how Spark works..

## 3. How Spark works:  
<img width="540" alt="spark-standalone-mode" src="https://user-images.githubusercontent.com/22542670/30005242-a22a0c5c-90fb-11e7-9d80-97efe540417f.png">

Above picture depicts Spark cluster.
- **Spark Master, Worker and Executor JVM’s:**
SparkMaster and Worker JVM’s are the resource managers. All worker JVM’s will register themselves with SparkMaster. They are very small. Example: Master use like 500MB of RAM & Worker uses like 1GB RAM.
- **Master:**
Master JVM’s job is to decide and schedule the launch of Executor JVM’s across the nodes. But, Mater will not launch executors. 
- **Worker:**
It is the Worker who heartbeats with Master and launches the executors as per schedule. 
- **Executor:**
Executor JVM has generic slots where tasks run as threads. Also, all the data needed to run a task is cached within Executor memory.
- **Driver:**
When we start our spark application with spark submit command, a driver will start wherever we do spark-submit and that driver will contact spark master to launch executors and run the tasks. Basically, Driver is a representative of our application and does all the communication with Spark.
<img width="473" alt="spark" src="https://user-images.githubusercontent.com/22542670/30005254-084b8f2e-90fc-11e7-8488-90033e632cc5.png">
- **Task:**
Task is the smallest unit of execution which works on a partition of our data. Spark actually calls them cores. —executor-cores setting defines number of tasks that run within the Executor. For example, if we have set —executor-cores to six, then we have six can run simultaneous threads within the executor JVM.
- **Resilience:**
Worker JVM’s work is only to launch Executor JVM’s whenever Master tells them to do so. If Executor crashes, Worker will restart it. If Worker JVM crashes, Master will start it. Master will take care of driver JVM restart as well. But then, if a driver restarts, all the Ex’s will have to restart. 
- **Distribution of CPU resources:**
By CPU resources, We are referring to the tasks/threads running within an executor. Let’s assume that the second machine has lot many more ram and cpu resources. Can we run more threads in this second machine? You can do that by tweaking spark-env.sh file and set SPARK_WORKER_CORES to 10  and the same setting if set to 6 in other machines. Then master will launch 10 threads/tasks in that second machine and 6 in the remaining one’s. But, u could still oversubscribe in general. SPARK_WORKER_CORES tells worker JVM as to how many cores/tasks it can give out to its underlying executor JVM’s.

## 4. Appendix:
Facebook came up with Corona to address this problem & leverage more CPU%. In their hadoop cluster, when FB was running 100’s of MR jobs with lots of them already in the backlog waiting to be run because all the MR slots were full with currently running MR jobs, they noticed that their CPU utilisation was pretty low (~60%). It was weird because they thought that all the M & R slots were full & they had a whole lot of backlog waiting out there for a free slot. What they notice was that in traditional MR, once a Map finishes, then TT has to let JT know that there’s empty slot. JT will then allot this empty slot to the next job. This handshake between TT & JT is taking ~15-20secs before the next job takes up that freed up slot. This is because, Heartbeat of JT is 3secs. SO, every 3secs, it checks with TT for free slots and also, its not necessary that the next job will be assigned in the very next heartbeat. So, FB added corona which is a more aggressive job scheduler added on top of JT. MR took 66secs to fill a slot while corona took like 55 secs (~17%). Slots here are M or R process id’s.  
With Spark, the slots inside the executor are generic. They can run M, R or join or a whole bunch of other kinds of transformations.
We get parallelism in Spark by having different threads running inside the executor. Basically, in Spark, we have one executor JVM on each machine. Inside this executor JVM, we’ll have slots. In those slots, tasks run. 
So spark threads vs MR processes for parallelism. 

## 5. Legend: 
- MR - MapReduce
- M - Map Job
- R - Reduce Job
- JT - JobTracker
- TT - TaskTracker
