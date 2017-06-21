## Distribution of Executors, Cores and Memory for a Spark Application running in Yarn:
In this blog, I'll cover how to size Executors, Cores and Memory while running spark application on Yarn. The three spark config params using which we can configure number of executors, cores and memory are:
- Number of executors (`--num-executors`)
- Cores per executor (`--executor-cores`) 
- Memory per executor (`--executor-memory`)

These three params play a very important role in spark performance as they control the amount of CPU & memory that spark application gets. This makes it very crucial for users to understand the right way to configure them. 

### Following list captures some recommendations to keep in mind while configuring them:
1. **Hadoop/Yarn/OS Deamons:** When we run spark application using a cluster manager like Yarn,  there’ll be several daemons that’ll run in the background like NameNode, Secondary NameNode, DataNode, JobTracker and TaskTracker. So, while specifying num-executors, we need to make sure that we leave aside enough cores (~1 core per node) for these daemons to run smoothly. 
2. **Yarn ApplicationMaster (AM):** ApplicationMaster is responsible for negotiating resources from the ResourceManager and working with the NodeManagers to execute and monitor the containers and their resource consumption. If we are running spark on yarn, then we need to budget in the resources that AM would need (~1024MB and 1 Executor).
3. **HDFS Throughput:** HDFS client has trouble with tons of concurrent threads. It was observed that HDFS achieves full write throughput with ~5 tasks per executor . So it’s good to keep the number of cores per executor below that number.
4. **MemoryOverhead:** Following picture depicts spark-yarn-memory-usage.   
![image](https://user-images.githubusercontent.com/22542670/27395274-de840270-56cc-11e7-8f3a-f78c4eecdac8.png)


Two things to make note of from this picture:
```markdown
 Full memory requested to yarn per executor =
          spark-executor-memory + spark.yarn.executor.memoryOverhead.
 spark.yarn.executor.memoryOverhead = 
        	Max(384MB, 7% of spark.executor-memory)
```
So, if we request 20GB per executor, AM will actually get 20GB + memoryOverhead = 20 + 7% of 20GB = ~23GB memory for us.
5. Running executors with too much memory often results in excessive garbage collection delays.
6. Running tiny executors (with a single core and just enough memory needed to run a single task, for example) throws away the benefits that come from running multiple tasks in a single JVM.

## Enough theory.. Let's go hands-on..
Now, let’s consider a 10 node cluster with following config and analyse different possibilities of executors-core-memory distribution:
**Cluster Config:**
```markdown
10 Nodes
16 cores per Node
64GB RAM per Node
```
### First solution: Tiny executors [One Executor per core]:  
Tiny executors essentially means one executor per core. Following will be the config with this approach:
- `--num-executors` = 16 x 10 = 160 (We've 10 node cluster with 16 cores per node. One exec per core means 16 executoers per node. A cluster of 10 such nodes means 16*10 = 160 executors)
- `--executor-cores`= 1 (one executor per core)
- `--executor-memory` = 64GB/16 = 4GB (Each node in the cluster has 64GB RAM. Number of executors per node = 16. So, memory allotted per executor is equal to 64GB/16 = 4GB)

**Analysis:** With only one executor per core, as we discussed above, we’ll not be able to take advantage of running multiple tasks in the same JVM. Also, shared/cached variables like broadcast variables and accumulators will be replicated in each core of the nodes which is **16 times**. Also, we are not leaving enough memory overhead for Hadoop/Yarn daemon processes and we are not counting in ApplicationManager. **NOT GOOD!**
### Fat executors - One Executor per node:
- Number of executors = 1 x 10 = 10
- Number of cores per executor = 16
- Memory per executor = 64GB


**Analysis:** With all 16 cores per executor, apart from ApplicationManager and daemon processes are not counted for, HDFS throughput will hurt and it’ll result in excessive garbage results. Also,**NOT GOOD!**

### What’s right config?
- Number of core per executors = 5 (for good HDFS throughput)
- Leave 1 core per node for Hadoop/Yarn daemons => num cores per node=16-1 = 15
- Total number of cores in cluster 15 x 10 = 150
```Number of executors = (total cores/cores per executor) = 150/5 = 30```
- Leaving 1 executor for AM => #executors = 29
- Number of # executors per node = 30/10 = 3
- Memory per executor = 64GB/3 = 21GB
- Counting off heap overhead = 7% of 21GB = 3GB. So, actual executor memory = 21 - 3 = 18GB

### Correct Answer: 
29 executors, 18GB memory each and 5 cores each!!
