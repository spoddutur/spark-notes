
#### Continuation from [part2](https://spoddutur.github.io/spark-notes/reb1)...

# Weaving a periodically changing cached-data with your streaming application...

### Problem Statement
In [part2](https://spoddutur.github.io/spark-notes/reb1), We saw how to keep-track of periodically changing cached-data. Now, we not only want to track it but also weave it (i.e., apply filter, map transformations) with our input dataframes or rdd's of our streaming application?

#### Nomenclature: I've coined cached-data alternatively as refdata or reference-data in this blog.

## Naive thoughts to handle this:
I've noticed people thinking of crazy ideas such as:
- Restart the Spark Context every time the refdata changes, with a new Broadcast Variable.
- Host the refdata behind a REST api and look-it-up via `foreachRDD or forEachPartition`.

## Spark-Native Solution:
I've proposed a spark-native solution to handle this case in much more sensible manner compared to the naive approaches mentioned above. Below TableOfContents walks you through the solution:
1. **Core idea:** Short briefing of the solution
2. **Demo:** Implementation of the solution
3. **Theory:** Discuss the theory to understand the workings behind this spark-native solution better.
4. **Conclusion:** Lastly, I’ll conclude with some key take-outs of the spark-native solutions discussed in [part2](https://spoddutur.github.io/spark-notes/reb1) and [part3](https://spoddutur.github.io/spark-notes/reb2).

## 1. Core Idea:
Per every batch:
a. We can broadcast the cached-data with any latest changes to it
b. Update our input using this broadcasted (latest) cached-data and 
c. Lastly unpersist/remove the broadcasted cache (not the cached data - but the braodcasted cache).
Repeat steps a through c per every batch

## Demo:
**How do we do this?**
```diff

// Generic function that broadcasts refData, invokes `f()` and unpersists refData.
def withBroadcast[T: ClassTag, Q](refData: T)(f: Broadcast[T] => Q)(implicit sc: SparkContext): Q = {
    val broadcast = sc.broadcast(refData)
    val result = f(broadcast)
    broadcast.unpersist()
    result
}

// transforming inputRdd using refData
def updateInput(inputRdd, broadcastedRefData): updatedRdd {
  inputRdd.mapPartition(input => {
   // access broadcasted refData to change input
  })
}

// invoking `updateInput()` using withBroadcast()
val updatedRdd = withBroadcast(refData) { broadcastedRefData =>
  updateInput(inputRdd, broadcastedRefData))
}

```
- Here, we wanted to transform `inputRdd` into `updatedRdd` but using periodically changing `refData`.
- For this, we declared an `updateInput() def where inputRdd is transformed into updatedRdd using inputRdd.mapPartition(..)`
- Now, invoke `withBroadcast() using 3 params: updateInput() as f(), refData and inputRdd`.
- That's it.. every new batch of inputRdd's in your streaming application will be transformed using the latest uptodate refData. This is because, we unpersist old refData as soon as the current batch of inputRdd is transformed.
- Also, note that in this approach workers will get latest refData per batch. Any updates to the refData done while a batch is being processed will not get broadcasted until the current batch finishes processing. Hence, any stale data lives only for the duration of one batch.
#### Essentially, as every new batch of input comes in, this helper function is transforming it using latest refData.

## Conclusion:
Some key takeouts of part1 and part2 solutions which are primarily spark-native (i.e., not using any external service):
-  Broadcast object is not Serializable and needs to be final. So, stop thinking about or searching for a solution to update it.
- So, **instead of thinking about updating a broadcasted refData, think of collecting changes to the reference data at one place i.e., at driver node and compute it in a distributed fashion.**
- If the requirement is to use the changing refData with your inputStream, then we now know how to handle this case bearing stale data for a maximum duration of one batch-interval. We've seen how the latest refData changes will be pulled once per every new batch of input points flowing into the system..
- Hopefully, this blog gave you a good insight into dealing with changing datapoints in your spark streaming pipeline.

So far, we looked at spark-native solutions to address the issue. Now, let's widen the horizon and look outside spark [here](https://spoddutur.github.io/spark-notes/reb3)

## Alternatives:
One can also think of converting the Reference Data to an RDD, then join it with input streams using either join() or zip() in such a way that we now have streaming Pair<MyObject, RefData>. 

[My HomePage](https://spoddutur.github.io/spark-notes/)

## References: 
- [Spark's Torrent Broadcast](https://github.com/apache/spark/pull/2217)
- [Spark ML-Lib streaming kmeans implementation using collect at driver approach](https://github.com/apache/spark/blob/master/mllib/src/main/scala/org/apache/spark/mllib/clustering/StreamingKMeans.scala)
- [Demo on weaving changing cluster centroids with input datapoints in kmeans](https://github.com/derrickburns/generalized-kmeans-clustering)
- [Spark thread on rebroadcating](http://apache-spark-user-list.1001560.n3.nabble.com/Broadcast-variables-can-be-rebroadcast-td22908.html)
