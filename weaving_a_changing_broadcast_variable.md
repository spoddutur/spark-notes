

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
But, if you requirement matches with the second one, then go ahead and continue reading..

Am proposing two approaches to handle this sensibly compared to the naive approaches mentioned above:
1. Convert the Reference Data to an RDD, then join the streams in such a way that I am now streaming Pair<MyObject, RefData>, though this will ship the reference data with every object.
2. Per every batch, we can unpersist the broadcast variable, update it and then rebroadcast it to send the new reference data to the executors

## DEMO time:
