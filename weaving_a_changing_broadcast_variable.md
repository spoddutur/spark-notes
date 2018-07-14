

### How to weave a periodically changing cached-data with your streaming application?

## Naive thoughts to handle this:
I've noticed people thinking of crazy ideas such as:
- Restart the Spark Context every time the refdata changes, with a new Broadcast Variable.
- Host the refdata behind a REST api and look-it-up via `foreachRDD or forEachPartition`.

#### Jargon: I've coined cached-data alternatively as refdata or reference-data in this blog.

## Checkpoint:
Before reading any further, its importance to check and understand your requirement:
- **First case:** Is it about caching/tracking the periodically changing reference-data? (VS)
- **Second case:** Is it about weaving (i.e., filter, map etc) the changing reference-data with input streaming data?

## First case: Caching the periodically changing reference-data
If your requirement is more inclined towards the first one, then I have discussed the solution to handle it in detail [here](https://spoddutur.github.io/spark-notes/rebroadcast_a_broadcast_variable)

## Second case:
But, if you requirement matches with the second one, then you are in the right place and continue reading..
Am proposing an approach to handle this case in much more sensible manner compared to the naive approaches mentioned above.

## DEMO time: Solution1
Per every batch, we can unpersist the broadcast variable, update input and then rebroadcast it to send the new reference data to the executors.

**How do we do this?**
```diff
def withBroadcast[T: ClassTag, Q](refData: T)(f: Broadcast[T] => Q)(implicit sc: SparkContext): Q = {
    val broadcast = sc.broadcast(refData)
    val result = f(broadcast)
    broadcast.unpersist()
    result
}

val updatedRdd = withBroadcast(refData) { broadcastedRefData =>
  updateInput(inputRdd, broadcastedRefData))
}

def updateInput(inputRdd, broadcastedRefData): updatedRdd {
  inputRdd.mapPartition(input => {
   // access broadcasted refData to change input
  })
}
```
- Here, we wanted to transform `inputRdd` into `updatedRdd` but using periodically changing `refData`.
- For this, we declared an `updateInput() def where inputRdd is transformed into updatedRdd using inputRdd.mapPartition(..)`
- Now, invoke `withBroadcast() using 3 params: updateInput() as f(), refData and inputRdd`.
- That's it.. every new batch of inputRdd's in your streaming application will be transformed using the latest uptodate refData. This is because, we unpersist old refData as soon as the current batch of inputRdd is transformed.
- Also, note that in this approach workers will not read the stale value after we change it.
#### Essentially, as every new batch of input comes in, this helper function is transforming it using latest refData.

## Alternatives:
One can also think of converting the Reference Data to an RDD, then join it with input streams using either join() or zip() in such a way that we now have streaming Pair<MyObject, RefData>. 

References: 
- https://github.com/apache/spark/pull/2217
- https://github.com/derrickburns/generalized-kmeans-clustering
- https://github.com/apache/spark/blob/master/mllib/src/main/scala/org/apache/spark/mllib/clustering/StreamingKMeans.scala

    
