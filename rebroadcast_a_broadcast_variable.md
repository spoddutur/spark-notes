### Did you ever thought of updating or re-broadcasting a broadcast variable?

## Why would we need this?
You have a stream of objects that you would like to filter based on some reference data.
This reference data will keep changing periodically
Dealing with such streaming applications which need a way to weave (filter, lookup etc) the streaming data using a changing reference data (from DB, files etc) has become a relatively common use-case.

## Is it only a relatively-common use-case?
I believe that this is more than a just being a relatively-common use-case in the world of ML applications or active-learning systems. Let me illustrate the situations which will help us understand their necessity:
- **Example1:** Consider a task of training k-means model given a set of data-points. After each iteration, one would want to update two things:
  - Update cluster centroids with new centres.
  - Reassign new cluster ids to input data-points based on new cluster centroids.
- **Example2:** Similarly, consider another example of phrase mining which aims at extracting quality phrases from a text corpus. A streaming application that is trying to do phrase-mining would want to have a:
A cache of the <mined phrases, their-term-frequency> across the worker nodes
Be able to update this cache as more phrases are mined.

## What is common in both these cases?
The reference data, be it the cluster centroids or the phrases mined, both the tasks would need to: 
1. Broadcast it to have a local cached copy per executor and 
2. Iteratively keep refining this broadcasted cache.
