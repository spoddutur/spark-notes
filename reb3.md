# Apache Ignite to hold data that needs rebroadcasting

### Continuation from [part-3](https://spoddutur.github.io/spark-notes/reb2)..

## Earlier Spark-Native solutions are more-or-less compromised work-arounds:
We’ve discussed in [part-2](https://spoddutur.github.io/spark-notes/reb1) and [part-3](https://spoddutur.github.io/spark-notes/reb2) different spark-native approaches to maintain a mutable distributed cache native-within-spark such as:
1. Hold the cache as globalCorpus at the driver which gets updated once per batch aggregating the localCorpus learnt from the current batch received.
2. Hold this cache as broadcast variable and Re-broadcast it periodically as it keeps changing.

But, all these approaches seem more or less like compromised work-arounds given the fact that RDD’s in spark are immutable. Let me elaborate more on this:
- Any data loaded in spark is either an RDD or Broadcast Variable or Accumulator.
- RDD and Broadcast variable are immutable.
- Accumulator are mutable. However, Spark natively supports only numeric type accumulators.
- So, any spark-native-solution attempting to hold mutable cache is more-or-less a workaround.

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

## Demo: TODO

## Conclusion: TODO

## References
- https://github.com/apache/ignite/blob/master/examples/src/main/java/org/apache/ignite/examples/datagrid/CacheQueryExample.java
- https://ignite.apache.org/features/datagrid.html
- https://ignite.apache.org/use-cases/caching/database-caching.html
