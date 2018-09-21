# Apache Ignite to hold data that needs rebroadcasting

### Continuation from [part-3](https://spoddutur.github.io/spark-notes/reb2)..

## Earlier Spark-Native solutions are more-or-less compromised work-arounds:
We’ve discussed in [part-2](https://spoddutur.github.io/spark-notes/reb1) and [part-3](https://spoddutur.github.io/spark-notes/reb2) different spark-native approaches to maintain a mutable distributed cache native-within-spark such as:
1. Hold the cache as globalCorpus at the driver which gets updated once per batch aggregating the localCorpus learnt from the current batch received.
2. Hold this cache as broadcast variable and Re-broadcast it periodically as it keeps changing.

But, both these approaches seem more or less like compromised work-arounds given the fact that RDD’s in spark are immutable. Let me elaborate more on this:
- Any data loaded in spark is either an RDD or Broadcast Variable or Accumulator.
- RDD and Broadcast variable are immutable.
- So, any spark-native-solution attempting to hold mutable cache in RDD or Broadcast form is more-or-less a workaround.
- Accumulators are mutable. However, Spark natively supports only numeric type accumulators.

## ApacheIgnite - A non-spark-native solution:

**Yes!** ApacheIgnite is an in-memory distributed SQL DB which comes with [database caching](https://ignite.apache.org/use-cases/caching/database-caching.html) as well.

- It enables users to keep the most frequently accessed data in memory.
- This cache data can be either partitioned or replicated across a cluster of computers.
- All of this cache can be setup native within your spark cluster.
- Note that any inserts are write-behind

Following figure illustrates the role of ApacheIgnite within our Spark application:
![image](https://user-images.githubusercontent.com/22542670/44307330-0c48a000-a3be-11e8-897d-a9fcd8def68a.png)

## Theory:
Ignite data can be exposed as mutable [IgniteRDD](https://github.com/apache/ignite/blob/master/modules/spark/src/main/scala/org/apache/ignite/spark/IgniteRDD.scala) in Spark which has benefits such as:
- It is a shared rdd i.e., all the spark-workers and spark-applications deployed in the same cluster can share the same state with igniteRDD.
- It is mutable because igniteRDD is nothing but a rather advanced API on top of your igniteData.
- The igniteRDD is already loaded in the cluster. So, there’s no re-loading/distributing the data to different executors of the cluster every time application starts.
- IGNITE SQL ENGINE faster than Spark SQL engine: Spark does support SQL queries. But, it doesn't do indexing i.e., every single query will end-up having full scan of data. Ignite has index. So, our sql queries will get executed in algorithmic time which is faster compared to the linear-time.  

**How does Updates/Ingestion happen to IgniteRDD:**
Ignite Streamers are streaming components, that ingest data in the fastest way possible into apache ignite. Hence, any streaming real-time cache-data updates/inserts happens fast.

## Demo: 
Here, I'll demo how to mine phrases in a spark streaming application and keep track of changing cache in ignite.
Note that, we saw solution for the same in part2 without using any external service.

```markdown
val sentencesDf = spark.read
   		       .format("text")
		       .load(“/tmp/gensim-input”).as[String]

val igniteWordsRDD = new IgniteContext[String, Int](sc, () => new IgniteConfiguration()).fromCache(“igniteWordsRDD")

// learn phrases from input
val phrasesRdd = sentencesDf.flatMap(sentence => Phrases.learnVocab(sentence))

// saving the phrase counts to ignite cache
igniteWordsRDD.saveValues(phrasesRdd)

// transform values in ignite cache - merge old phraseRecords and new phraseRecords
igniteWordsRDD.groupBy(“phrase”).agg(sum($”count”))
```

## Conclusion: Analysis and Advantages:
- **Still thinking as to how above solution is easing user off any burden compared to spark-native solutions?**
  - In Spark native solution to keep track of mined phrases:
  	1. We maintained a global cache at driver
	2. We also maintained a per partition cache and collected the new mined phrases per partition in every batch here
	3. Merge partition cache's into our global cache at driver.
  - Its all happening so seamlessly here in 2 steps - `igniteRDD.saveValue() and igniteRDD.groupBy("phrase").agg(..)`:
      - The first step is to save new phrases learnt per batch in ignite using `saveValues()`.
      - It basically saves values from given RDD into Ignite and generates a unique key for each value of the given RDD.
      - The second step is to aggregating phrase counts within ignite. That's it!!
  
- **Weaving has also become easy**
  - Because our ignite cache is an RDD, to weave the phrases vocab with our input rdd's is so much easier. It feels like home for spark where-in it boiled down to rdd-to-rdd weaving!!! 

- **Better performant scans than spark**
  - Any query on igniteRdd's will be faster than spark scans.
  - SparkRDD is not indexed. Consequently, any queries on it will scan the entire data. 
  - IgniteRDD maintains index. Hence, any query here will be done much faster!!!
  - Moreover, imagine if we load data bigger than our memory, it'll naturally spill to disk. Consequently, any queries on such big SparkRDD's will constantly have to do **data-spill-to-and-from-memory-and-disk inorder to scan the entire RDD**.

- **No data collection is happening at the driver.**
This could potentially impose the amount of data one can cache i.e., ideally, `collect()` on big datasets is not recommended.

- **Cross-Application sharing of the cache in real-time*
	- Another big perk with ignite solution is that, as our phrase mining application is learning new phrases and saving them in ignite real-time, any other down stream applications that needs this phrases vocabulary can get the latest cached vocab seamlessly
	- Just register igniteContext and load the cache. That's it!!

Key Takeouts:
1. Now, hopefully, you would agree with me and stop thinking of modifying a broadcast variable.
2. Categorize your need to broadcast into 
	1. Only collecting cache data as we see more and more batches
	2. Collect cache data nd also use the latest cached data within our application.
3. We saw how to solve solutions to both the above scenarios without relying any external service. It does computation in a distributed fashion and collects changes at one place i.e., driver.
4. Spark-native approaches are simple for cases where cache-size is small or its some quick POC todo
5. Else, as we have seen in this article, using an external service like ApacheIgnite eases user off quite some payload and brings on table some other benefits like seamless weaving with RDD's and cross-application cache sharing.
6. With Ignite, we saw how both computation and storage is happening in a distributed fashion.

## References
- https://github.com/apache/ignite/blob/master/examples/src/main/java/org/apache/ignite/examples/datagrid/CacheQueryExample.java
- https://ignite.apache.org/features/datagrid.html
- https://ignite.apache.org/use-cases/caching/database-caching.html
