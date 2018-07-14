

### Do you have a requirement to weave a periodically changing reference-data/cached-data with your streaming application?

## Naive thoughts to handle this:
I've noticed people thinking of crazy ideas such as:
- Move the reference data lookup into a forEachPartition or forEachRdd so that it resides entirely on the workers. However the reference data lives beind a REST API so I would also need to somehow store a timer / counter to stop the remote being accessed for every element in the stream.
- Restart the Spark Context every time the refdata changes, with a new Broadcast Variable.

Before reading any further, Its importance to check with yourself if your requirement is:
- Having a need to track and hold-on to the updates on reference-data (VS)
- Having the need to weave (i.e., filter or map etc) the changing reference-data with input streaming data

If the requirement is the first one, then I have discussed the solution to handle it in detail [at part1 of this blog]
But, if the requirement atches with the second one, then go ahead and continue reading..

## Two solutions am proposing to handle this better than above mentioned naive approaches:
1. Convert the Reference Data to an RDD, then join the streams in such a way that I am now streaming Pair<MyObject, RefData>, though this will ship the reference data with every object.
2. Per every batch, we can unpersist the broadcast variable, update it and then rebroadcast it to send the new reference data to the executors

## DEMO time:
