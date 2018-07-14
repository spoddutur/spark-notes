

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

## Second case: Two solutions
But, if you requirement matches with the second one, then you are in the right place and continue reading..
Am proposing two approaches to handle this case in much more sensible manner compared to the naive approaches mentioned above:
1. Convert the Reference Data to an RDD, then join the streams in such a way that I am now streaming Pair<MyObject, RefData>, though this will ship the reference data with every object.
2. Per every batch, we can unpersist the broadcast variable, update it and then rebroadcast it to send the new reference data to the executors

## DEMO time:

```diff
def withBroadcast[T: ClassTag, Q](refData: T)(f: Broadcast[T] => Q)(implicit sc: SparkContext): Q = {
    val broadcast = sc.broadcast(refData)
    val result = f(broadcast)
    broadcast.unpersist()
    result
}
```
Let's discuss what we did in the above code?
- We created a helper function withBroadcast() that does:
  - Takes in an refData of type `T`.
  - Broadcast refData.
  - Transforms input using `f: Broadcast[T] => Q)` and get the result `Q`.
  - Once the transformation is done, we remove or unpersist the broadcasted refData
  - Finally, return the result
  - Essentially, as every new batch of input comes in, this helper function is transforming it using latest refData.
  
Let's see how to use it?
```diff
+ use withBroadcast() to broadcast refData inorder to transform input rdd
val updatedRdd = withBroadcast(refData) { broadcastedRefData =>
  updateInput(inputRdd, broadcastedRefData))
}

def updateInput(inputRdd, broadcastedRefData): updatedRdd {
  inputRdd.mapPartition(input => {
   // access broadcasted refData to change input
  })
}
```
- Here, we wanted to transform inputRdd into updatedRdd but using periodically changing refData
- For this, we declared an updateInput() def where inputRdd is transformed into updatedRdd using inputRdd.mapPartition(..)
- Now, invoke withBroadcast() using updateInput() as f(), refData and inputRdd params.
- That's it.. every new batch of inputRdd's in your streaming application will be transformed using the latest uptodate refData. This is because, we unpersist old refData as soon as the current batch of inputRdd is transformed.

