### Did you ever thought of updating or re-broadcasting a broadcast variable?

## Why would we need this?
- You have a stream of objects that you would like to filter based on some reference data.
- This reference data will keep changing periodically.
- One would typically think of broadcasting the reference data to give every executor its own local cached-copy. But then, how to handle periodical updates to this? This is where perhaps the thought of having an updatable broadcast or rebroadcasting gets instilled in user's mind.

Dealing with such streaming applications which need a way to weave (filter, map etc) the streaming data using a changing reference data (from DB, files etc) has become a relatively common use-case.

## Is this requirement only a relatively-common use-case?
I believe that this is more than just being a relatively-common use-case in the world of `Machine Learning` applications or `Active Learning` systems. Let me illustrate the situations which will help us understand this necessity:
- **Example1:** Consider a task of training k-means model given a set of data-points. After each iteration, one would want to update two things:
  - Update cluster centroids with new centres.
  - Reassign new cluster ids to input data-points based on new cluster centroids.
- **Example2:** Similarly, consider another example of phrase mining which aims at extracting quality phrases from a text corpus. A streaming application that is trying to do phrase-mining would want to have:
	- Cache of the `<mined-phrases, their-term-frequency>` across the worker nodes.
	- Be able to update this cache as more phrases are mined.

## What is common in both these cases?
The reference data, be it the cluster centroids or the phrases mined, in both the tasks would need to: 
1. Broadcast it to have a local cached copy per executor and 
2. Iteratively keep refining this broadcasted cache.

#### For the cases discussed above, one would think that we want a way to broadcast our periodically changing reference data.  But is it really needed? Let’s see alternative perspectives in which we can think to handle such cases.

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
Notice `phrasesBc.value.addVocab()` code written above. This is trying to update broadcasted variable which will run fine in local run. But, in cluster, this doesn’t work because:
- The `phrasesBc` broadcasted value is not a shared variable across the executors running in different nodes of the cluster.  
- One has to be mindful of the fact that `phrasesBc` is indeed a local copy one in  each of the cluster’s worker nodes where executors are running. 
- Therefore, changes done to `phrasesBc` by one executor will be local to it and are not visible to other executors.

### One solution am proposing:
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
val minedPhrases = new HashMap[String, Int]()

sentencesDf.mapPartitions(sentencesInthisPartitionIterator => {
      
     // word and its count map
     val partitionCorpus = new HashMap[String, Int]() 
      while (sentencesInPartitionIter.hasNext) {
	val sentence = sentencesInPartitionIter.next()
	val sentenceCorpus: HashMap[String, Int] = Phrases.learnVocab(sentence)
	
	// merge sentence corpus with partition corpus
	sentenceCorpus.foreach(x => partitionCorpus.put(x._1, x._2))	
       }
}).groupBy($”key”).agg(sum($”value”)).collect().foreach(x => minedPhrases.put(x))
```

### What did we achieve by this
- Driver machine is the only place where we maintain a cumulative corpus of phrases learnt
- At the same time, this approach doesnt overload driver with periodic updates one per record.
- Driver copy will get updated only once per batch.
- Phrase mining workload is shared beautifully across all the executors.
- This way, every time we receive a new batch of input data points, the reference data i.e., phrases gets updated only at one place i.e., driver node and at the same time learning corpus is done in a distributed way.


