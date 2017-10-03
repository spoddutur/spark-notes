# Knowledge Browser

## Problem Statement:
#### Despite being the first class citizen in Spark holding the key corporate asset (i.e., data), Datasets are not getting enough attention it needs in terms of making them searchable.

I wrote this blog to help my self understand how to make datasets available for outside world a.k.a how to make them searchable. I hope it helps you as much it helped me. In this blog post, we’ll see:
1. How can we make datasets searchable 
  - esp. without using any search engine
2. Same data, different representations
  - Rearrange the same data in different schema’s 
  - Understand the impact of data schema on search time.
3. Conclusion - Performance testing
- Evaluate each of these approaches and see which is more appropriate way to structure the datasets when it comes to making them searchable.

## 1. How can we make datasets searchable
Please refer to my git repository ![here](https://github.com/spoddutur/graph-knowledge-browser) where I’ve provided a working example of how to make your knowledge searchable on top of ApacheSpark.

## 2. Same data, different representations:
I've used ~4 million (100MB) countries profile information as knowledge base from world bank open data (http://data.worldbank.org) in this project. Following table displays some sample rows:
<img width="873" alt="screen shot 2017-08-03 at 11 45 41 pm" src="https://user-images.githubusercontent.com/22542670/28936625-e4cf32e4-78a5-11e7-99f6-cdec6b93ce71.png">

 I tried different ways to structure this data and evaluated their performance using simple search_by_country_id query.

### 2.1 Data Frames:
The very first attempt to structure the data was naturally the simplest of all  i.e., DataFrames, where all the information of the country is in one row.

**Schema:**
![image](https://user-images.githubusercontent.com/22542670/31138340-bccdb67e-a88b-11e7-9063-e49fb4cde06a.png)

#### Query by CountryId response time: 100ms

### 2.2 RDF Triplets
Next, I represented the data as ![RDF triplets](https://en.wikipedia.org/wiki/RDF_Schema). In this approach, we basically, take each row and convert it into triplets of ```markdown Subject, Predicate and Object``` as shown in the table below:
![image](https://user-images.githubusercontent.com/22542670/31138372-d8b7ade0-a88b-11e7-9056-ef7612282ed3.png)

**Schema:**
![image](https://user-images.githubusercontent.com/22542670/31138377-dd382ef8-a88b-11e7-82bf-56d243e618c3.png)

#### Query by CountryId search: 6501ms

### 2.3 RDF Triplets as linked data
Next, I represented the same data as RDF triplets of linked data. The only difference between earlier approach and this one is that, the Subject here is linked to a unique id which inturn holds the actual info as shown below:
![image](https://user-images.githubusercontent.com/22542670/31138384-e7386940-a88b-11e7-91a1-e44fa2c4ee60.png)

**Schema:**
![image](https://user-images.githubusercontent.com/22542670/31138389-edefd58e-a88b-11e7-9f11-e97963793765.png)

#### Query by country id search: 25014ms

### 2.4 Graph Frames
Next I tried to structure the data as graph with vertices, edges, and attributes. Below pic gives an idea of how country info looks like in this approach:

![image](https://user-images.githubusercontent.com/22542670/31138409-ff0178f0-a88b-11e7-8ca9-8b7306b60278.png)

**Schema:**
![image](https://user-images.githubusercontent.com/22542670/31138431-16117fea-a88c-11e7-95d4-bdea35e3ed84.png)

#### Number of Vertices: 4278235
#### Number of Edges:15357957
#### Query by countryId response time - 7637ms

## 3. Observations:
- Lesser the links, better the performance
- When the nature of your data is homogenous, capturing all the information of a record in a single row gives the best performance in terms of search time.
- If you are dealing with heterogenous data, where all the entities cannot share a generic schema, then simple RDF triplets yields better response times compared to LinkedData RDF triplets representation.
- Though higher response time is a downside of LinkedData, it is the recommended approach to connect, expose and make your data available for semantic search. (http://linkeddata.org/)
- GraphFrames is very intuitive for user to structure the data in many cases. Its response times are comparable to RDF triplets search and they also open up doors to exploit graph algorithms like triangle count, connected commonents, BFS etc

