# Knowledge Browser
**Problem Statement:** Despite being the first class citizen in Spark, holding the key corporate asset i.e., data, Datasets are not getting enough attention it needs in terms of making them searchable.

### Cover Points:
In this blog, we'll look at following items:
1. **How can we make datasets searchable** 
  - esp. without using any search engine
  - SQL like queries for search
2. **Same data, different representations**
  - Rearrange the same data in different schema’s 
  - Understand the impact of data schema on search time.
3. **Conclusion - Performance testing**
- Evaluate each of these approaches and see which is more appropriate way to structure the datasets when it comes to making them searchable.

## 1. How can we make datasets searchable
I’ve implemented a working sample application using worldbank open data that does the following:
- A UI Dashboard built on top of Spark to browse knowledge (a.k.a data) 
- Real-time query spark and visualise it as graph.
- Supports SQL query syntax.
- This is just a sample application to get an idea on how to go about building any kind of analytics dashboard on top of the data that Spark processed. One can customize it according to their needs.

## Demo
![out](https://user-images.githubusercontent.com/22542670/28935434-31a77148-78a2-11e7-97d6-3267f3cd2b16.gif)

Please refer to my git repository [here](https://github.com/spoddutur/graph-knowledge-browser) for further details. 

## 2. Same data, different representations:
I've used ~4 million countries profile information as knowledge base from [world bank open data](http://data.worldbank.org) in this project. Following table displays some sample rows:
<img width="873" alt="screen shot 2017-08-03 at 11 45 41 pm" src="https://user-images.githubusercontent.com/22542670/28936625-e4cf32e4-78a5-11e7-99f6-cdec6b93ce71.png">

I tried different ways to structure this data and evaluated their performance using simple *```search by country id```* query.
Let's jump in and take a look at what are these different schema's that I tried and how they performed when queried by CountryId...

### 2.1 Data Frames:
The very first attempt to structure the data was naturally the simplest of all  i.e., DataFrames, where all the information of the country is in one row.

**Schema:**

![image](https://user-images.githubusercontent.com/22542670/31139612-b5093522-a88f-11e7-8b7d-65ad7e68c5f3.png)

**Query by CountryId response time:** 100ms

### 2.2 RDF Triplets
Next, I represented the same data as ![RDF triplets](https://en.wikipedia.org/wiki/RDF_Schema). In this approach, we basically, take each row and convert it into triplets of ```Subject, Predicate and Object``` as shown in the table below:


<img width="500" src="https://user-images.githubusercontent.com/22542670/31138372-d8b7ade0-a88b-11e7-9056-ef7612282ed3.png">


**Schema:**

<img width="300" src="https://user-images.githubusercontent.com/22542670/31138377-dd382ef8-a88b-11e7-82bf-56d243e618c3.png">

**Query by CountryId response time:** 6501ms

### 2.3 RDF Triplets as LinkedData
Next, I represented the same data as RDF triplets of linked data. The only difference between earlier approach and this one is that, the Subject here is linked to a unique id which inturn holds the actual info as shown below:

<img width="600" src="https://user-images.githubusercontent.com/22542670/31138384-e7386940-a88b-11e7-91a1-e44fa2c4ee60.png">

**Schema:**

<img width="300" src="https://user-images.githubusercontent.com/22542670/31139686-ec1dfb1a-a88f-11e7-9895-a518f812cbbf.png">

**Query by CountryId response time:** 25014ms

### 2.4 Graph Frames
The last attempt that I tried was to structure the data as graph with vertices, edges, and attributes. Below picture gives an idea of how country info looks like in this approach:

<img width="500" src="https://user-images.githubusercontent.com/22542670/31138409-ff0178f0-a88b-11e7-8ca9-8b7306b60278.png">

**Schema:**

<img width="300" src="https://user-images.githubusercontent.com/22542670/31139714-026644fe-a890-11e7-828a-acb7a5e2e0c2.png">

- **Number of Vertices:** 4,278,235
- **Number of Edges:** 15,357,957
- **Query by CountryId response time:** 7637ms

## 3. Conclusion:
I wrote this blog to demonstrate:
- **How to make datasets searchable** and
- **The impact of data schema on search response time.** 

For this, I tried different ways to structure the data and evaluated its performance. I hope it helps you and gives a better perspective in structuring your data for your Spark application.

### 3.1 Key takeouts:
- When the nature of your data is homogenous, capturing all the information of a record in a single row gives the best performance in terms of search time.
- If you are dealing with heterogenous data, where all the entities cannot share a generic schema, then simple RDF triplets yields better response times compared to [LinkedData RDF Triplets](http://linkeddata.org/) representation.
- Though higher response time is a downside of [LinkedData](http://linkeddata.org/), it is the recommended approach to connect, expose and make your data available for semantic search.
- GraphFrames is very intuitive for user to structure the data in many cases. Its response times are comparable to RDF triplets search and they also open up doors to exploit graph algorithms like triangle count, connected commonents, BFS etc

### 3.2 Note:
The search query here, essentially filters the datasets and returns the results i.e., it is a filter() transformation applied on data. So, the observed response times per schema not only applies to search but it also applies to any transformations that we apply on spark data. **This experiment definetely tells us how big is the impact of dataschema on the performance of your spark application. Happy data structuring !!**
