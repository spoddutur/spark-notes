### Did you ever thought of updating or re-broadcasting a broadcast variable?

## Why would we need this?
- You have a stream of objects that you would like to filter based on some reference data.
- This reference data will keep changing periodically.
- One would typically think of broadcasting the reference data to give every executor its own local cached-copy. But then, how to handle periodical updates to this? This is where perhaps the thought of having an updatable broadcast or rebroadcasting gets instilled in user's mind.

Dealing with such streaming applications which need a way to weave (filter, map etc) the streaming data using a changing reference data (from DB, files etc) has become a relatively common use-case.

## Is this requirement only a relatively-common use-case?
I believe that this is more than just being a relatively-common use-case in the world of `Machine Learning` applications or `Active Learning` systems. Let me illustrate the situations which will help us understand this necessity:
- **Example1:** Consider a task of training k-means model given a set of data-points. After each iteration, one would want to have:
	- Cache cluster-centroids 
	- Be able to update this cached centroids after each iteration
- **Example2:** Similarly, consider another example of phrase mining which aims at extracting quality phrases from a text corpus. A streaming application that is trying to do phrase-mining would want to have:
	- Cache of the `<mined-phrases, their-term-frequency>` across the worker nodes.
	- Be able to update this cache as more phrases are mined.

## What is common in both these cases?
The reference data, be it the cluster centroids or the phrases mined, in both the tasks would need to: 
1. Broadcast it to have a local cached copy per executor and 
2. Iteratively keep refining this broadcasted cache.
3. Most importantly, the reference-data that we are learning/mining is very small.

#### For the cases discussed above, one would think that we want a way to broadcast our periodically changing reference data.  But, given that such cases have very small sized reference data, is it really needed to have a local copy per executor? Let’s see alternative perspectives in which we can think to handle such cases.

## Why should we not think of workarounds to update broadcast variable?
Before going further into alternative perspectives, please do note that the Broadcast object is not Serializable and needs to be final. So, stop thinking about or searching for a solution to update it.

## Demo time: Right perspective/approach to handle it:
Now, hopefully, you are also in the same page as me to stop thinking of modifying a broadcast variable. Let's explore the right approach to handle such cases. Enough of talk. Let's see the code to demo the same.

### Demo:
Consider phrase mining streaming application. We want to cache mined-phrases and keep refining it periodically. How do we do it at large scale?

### Common mistake:
```markdown
    # init phrases corpus as broadcast variable
    val phrasesBc = spark.sparkContext.broadcast(phrases)
    
    # load input data as dataframe
    val sentencesDf = spark.read
   			      .format("text")
			      .load("/tmp/gensim-input").as[String]
			      
    # foreach sentence, mine phrases and update phrases vocabulary.
    sentencesDf.foreach(sentence => phrasesBc.value.updateVocab(sentence))
```

### Above code will run fine in local, but not in cluster..
Notice `phrasesBc.value.updateVocab()` code written above. This is trying to update broadcasted variable which will run fine in local run. But, in cluster, this doesn’t work because:
- The `phrasesBc` broadcasted value is not a shared variable across the executors running in different nodes of the cluster.  
- One has to be mindful of the fact that `phrasesBc` is indeed a local copy one in  each of the cluster’s worker nodes where executors are running. 
- Therefore, changes done to `phrasesBc` by one executor will be local to it and are not visible to other executors.

### How to solve this without broadcast?:
- Our streaming input data is split into multiple partitions and passed over to multiple executor nodes.
- Mine `<phrase, phrase-count>` info as RDD's or Dataframes locally for each partitioned block (i.e., locally within that executor).
- Combine the mined rdd phrases per partition using `aggregateByKey or combineByKey` to sum up the `phrase-count's`.
- Collect the aggregated rdd phrases at driver !!

### Code
```markdown
val sentencesDf = spark.read
   		       .format("text")
		       .load(“/tmp/gensim-input”).as[String]

// word and its term frequencies
val globalCorpus = new HashMap[String, Int]()

// learn local corpus per partition
val partitionCorpusDf = sentencesDf.mapPartitions(sentencesInthisPartitionIterator => {
      
     // 1. local partition corpus
     val partitionCorpus = new HashMap[String, Int]() 
     
     // 2. iterate over each sentence in this partition
     while (sentencesInPartitionIter.hasNext) {
     
	val sentence = sentencesInPartitionIter.next()
     
        // 3. mine phrases in this sentence 
        val sentenceCorpus: HashMap[String, Int] = Phrases.learnVocab(sentence)
	
	// 4. merge sentence corpus with partition corpus
	val partitionCount = partitionCorpus.getOrElse(x._1, 0)
	sentenceCorpus.foreach(x => partitionCorpus.put(x._1, partitionCount + x._2))	
     }
})

// 5. aggregate partition-wise corpus into one and collect it at the driver.
// 6. finally, update global corpus with the collected info.
partitionCorpusDf.groupBy($”key”)
		.agg(sum($”value”))
		.collect()
		.foreach(x => {
			// merge x with global corpus
			val globalCount = globalCorpus.getOrElse(x.word, 0)
			val localCount = x.count
			globalCorpus.put(x.word, globalCount+localCount)
		})
```

### What did we achieve by this
- Driver machine is the only place where we maintain a cumulative corpus of phrases learnt
- At the same time, this approach doesnt overload driver with updates of one per record.
- Driver copy will get updated only once per batch.
- Phrase mining workload is shared beautifully across all the executors.
- Essentially, every time we receive a new batch of input data points, the reference data i.e., phrases gets updated only at one place i.e., driver node. Also, at the same time, the job of mining phrases is computed in a distributed way.

### FAQ
1. **Why are we collecting reference-data at the driver?**
	- One might have apprehensions on collecting the reference-data at the driver.
	- But note that, in these usecases, the reference data being collected at driver is a small cache.
	- Also, every time we are collecting only a small set of new data to be added to this cache.
	- Just make sure that the driver machine has enough memory to hold the reference data. That's it!! 

2. **Why have globalCorpus in driver. Cant we maintain this using an in-memory alternatives like Redis?**
	- Yes! One use an in-memory cache for this purpose.
	- But, be mindful that with spark distributed computing, every partition run by every executor will try to update its local count simultaneously. 
	- So, if you want to take this route, then make sure to take up the additional responsibility of lock-based or synchronous updates to cache.
	
3. **Why not use Accumulators for globalCorpus?**
	- Accumulators work fine for mutable distributed cache.
	- But, Spark natively supports accumulators of numeric types.
	- The onus is on programmers to add support for new types by subclassing [AccumulatorV2](https://spark.apache.org/docs/2.2.0/api/scala/index.html#org.apache.spark.util.AccumulatorV2) as shown [here](https://spark.apache.org/docs/2.2.0/rdd-programming-guide.html#accumulators).

4. **Can we write the reference data directly into a file instead of collecting it in a globalCorpus as shown in code below?**
	```
	// write aggregated counts per batch in a file
	partitionCorpusDf.groupBy($”key”)
		.agg(sum($”value”))
		.foreach(x => {
			// write x to file
		})
	```
	- Above code is writing the aggregated counts to file. But, note that this is writing aggregated counts per batch. So, if we receive the word `"apache spark"` once in batch1 and later once more in batch2, then this approach will write `<"apache spark", 1>` entry in the file twice. This approach is basically agnostic of the counts aggregated in earlier batches.
	- So, there's a need to merge the batch-wise counts in the global corpus.
	- Save the globalCorpus into a file or some database using a shutdown hook right before our streaming application shuts down.

### Key Takeouts
Hopefully, this article gave you a perspective to:
- Not think about updating broadcast variable
- Instead, think of collecting changes to the reference data at one place i.e., at driver node and compute it in a distributed fashion.

So far, we've seen how to learn and eventually persist a periodically changing reference-data or global-dictionary that keep on evolving as we see more and more data like in active learning systems. Now, if you still have a requirement to weave this changing reference-data with your input stream? Then, check out my blog on [Weaving a changing broadcast variable with input stream](https://spoddutur.github.io/spark-notes/weaving_a_changing_broadcast_variable) where am going to demo ways to do it..

[My HomePage](https://spoddutur.github.io/spark-notes/)
