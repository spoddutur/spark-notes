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

## Solutions:
In this blog, am going to discuss how to handle following listed requirements:
1. How to keep track of periodically changing ref-data as more and more data is seen.
2. How to keep track of periodically changing ref-data + weave it with our streaming application.

I've categorized and discussed the solutions for the above listed requirements here: 
### 1. Spark-Native Solutions:
Here, I'll demo how to handle the two problems at hand within Spark.
### 2. Outside-Spark Solutions:
Here, We'll make use of external services outside spark to achive the same.

Hope this helps!!
