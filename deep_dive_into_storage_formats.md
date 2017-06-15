## Apache Spark - Deep Dive into Storage Format's
Apache Spark has been evolving at a rapid pace, including changes and additions to core APIs. Spark being an in-memory big-data processing system, memory is a critical indispensable resource for it. So, efficient usage of memory becomes very vital to it. Let’s try to find answer to following questions in this article:

- What storage format did Spark use?
- How did storage format evolve over a period of time?

### What storage format did Spark use?
- **Spark 1.0 to 1.3:** It started with RDD’s where data is represented as Java Objects.
- **Spark 1.4 to 1.6:** Deprioritised Java objects. DataSet and DataFrame evolved where data is stored in row-based format.
- **Spark 2.x:** Support for Vectorized Parquet which is columnar in-memory data is added.

![image](https://user-images.githubusercontent.com/22542670/26998244-18ed1b40-4d9f-11e7-897d-5d2b032c3001.png)


### How did storage format evolve over a period of time?
The answer to this question is not only interesting but also lengthy. Let’s dive into the driving factors and the journey of how Storage formats evolved in two parts:
1. Advancement from RDD to Row-based Dataset.
2. From Row-based Dataset to Column-based Parquet.

## Part1: Evolution of Data Storage Format's - Advancement of RDD to Row-based Dataset.

We cannot find the answer for this question, without diving into Project Tungsten. Just go with the flow and you’ll reason out as to why are we looking into Tungsten soon. Project Tungsten has been the largest change to Spark’s execution engine since its inception. This project has led to some fundamental changes in Spark. One such change gave birth to DataSet. Let’s start by looking at the goal of Project Tungsten.
## Objective of Project Tungsten 
![Image](https://user-images.githubusercontent.com/22542670/26983352-183c0c06-4d59-11e7-8fd9-a4d0cfd3c1d1.png)


### Why this objective?
The focus on CPU efficiency is motivated by the fact that Spark workloads are increasingly bottlenecked by CPU and memory use rather than IO and network communication. To understand this better:
- Let’s see the hardware trends over the past 7 years. Looking at the graph, its obvious that both DISK I/O and Network I/O have improved  10x faster:
![Image](https://user-images.githubusercontent.com/22542670/26983390-3aaa2606-4d59-11e7-93e9-a8c4db193964.png)

- Another trend that’s noticed was, in Spark’s shuffle subsystem, serialisation and hashing (which are CPU bound) have been shown to be key bottlenecks, rather than raw network throughput of underlying hardware.

These two trends mean that Spark today is constrained more by CPU efficiency and memory pressure rather than IO

### Problem1:
First things first..We need to understand the problem with old execution engine before we think of solutions to improve efficiency. To better understand this, let’s take a simple task of filtering a stream of integers and see how spark 1.x used to interpret it.
![Image](https://user-images.githubusercontent.com/22542670/26983465-7fd4819a-4d59-11e7-8f4b-4d5ee63b76de.png)

### What is the problem with this query plan?
As mentioned in the picture above, Spark doesn't know:
- What operation user is trying to perform on data? and
- What is the type of data?

Because of the lack of transparency, there’s barely any scope for spark to optimise the query plan.

### What transparency does spark need?
Spark needs transparency in 2 aspects:
1. **Data Schema:**
- How many and what fields are there in each data record?
- What are the datatypes of each field?
2. **User operation:** Spark should know what kind of operation user is trying to perform on which field of the data record.

### How does this transparency help Spark improve performance?
Its easier to illustrate it with an example. Consider joining two inputs `df1` and `df2`. 
JoinCondition: ```column x of df1 = column y of df2.```

Following figure depicts the join and its query plan that spark generates:
![Image](https://user-images.githubusercontent.com/22542670/26983475-8bf4d308-4d59-11e7-9a53-ffc48f712fa1.png)
Because the join condition is an anonymous function (`myUDF`), the only way for spark to perform this join is:
1. Perform cartesian product of df1 with df2.
2. Now, apply filter fn (myUDF) on the cartesian result which is an anonymous function that filters with df1[x] == df2[y] condition. Runtime = n^2!!

### How can transparency help spark improve this?
The problem in the above query was that join condition is an anonymous function. If spark is aware of the join condition and data schema [datatype of x and y columns] then spark would:
First sort df1 by x 
Next sort df2 by y
SortMerge Join df1 and df2 by df1[x] == df2[y]!! Runtime reduced to  nlogn!!
![Image](https://user-images.githubusercontent.com/22542670/26983488-9872b8b6-4d59-11e7-9562-661f7b9e305e.png)


### Action Plan1:
#### Have user register data schema.

### Problem2:
Spark tries to do everything in-memory. So, the next question is to know if there is a way to reduce memory footprint. We need to understand how is data laid out in memory for this.

### How is data laid out in memory? 
With RDD’s data is stored as Java Objects. There’s a whole lot of serialisation, deserialisation, hashing and object creation that happens whenever we want to perform an operation on these java objects’s and during shuffles. Apart from this Java objects have large overheads. 

### Should spark stop relying on JavaObjects? Why?
Ans: Java Objects have large overheads
Consider a simple string “abcd”. One would think it would take about ~4bytes of memory. But in reality, a java string variable storing the value “abcd” would take 48 bytes. Its breakdown is shown in the picture below. Now, imagine the amount of large overheads a proper JavaObject like a tuple3 of (Int, String, String) shown on the right hand side of the picture below takes.
![Image](https://user-images.githubusercontent.com/22542670/26983508-a7bc2d5c-4d59-11e7-8d51-18ae60cd9ef0.png)
### Action Plan2: 
#### Instead of java objects, come up with a data format which is more compact and less overhead

### Action Plan1 + Action Plan2 together:
Have user register data schema.
Create new data layout which is more compact and less overhead.
#### This paved way to “Dataframes and Datasets”

### What is DataSet/DataFrame?
A Dataset is a strongly-typed, immutable collection of objects with 2 important changes that spark introduced as we discussed in our action plan:
- Introduced new Binary Row-Based Format
- Data Schema Registration

**Introduced new Binary Row-Based Format:**
- To avoid the large overheads of a java object, Spark adapted new binary row-based storage as described in the table below:

![Image](https://user-images.githubusercontent.com/22542670/26983544-be7f6cf2-4d59-11e7-9398-47c5b0959b9d.png )

Following picture illustrates the same with an example. In this example, we took a tuple3 object **(123, “data”, “bricks”) and** and let’s see how its stored in this new row-format.  
![Image](https://user-images.githubusercontent.com/22542670/26983560-cc595054-4d59-11e7-805e-c3526ca4d38e.png)
- The first field `123` is stored in place as its primitive. 
- The next 2 fields `data` and `bricks` are strings and are of variable length. So, an offset for these two strings is stored in place [`32L` and `48L` respectively shown in the picture below]. 
- The data stored in these two offset’s are of format “length + data”. At offset 32L, we store `4 + data` and likewise at offset 48L we store `6 + bricks`.

### Data Schema Registration
Following example shows how to register data schema:
```markdown
case class University(name: String, numStudents: Long, yearFounded: Long)
val schools = spark.read.json(“/schools.json").as[University]
```
**Advantage**
1. So, our data is directly read as instances of university object
2. Spark provides Encoder API for DataSet’s which is responsible for converting to and from spark internal Tungsten binary format.
3. Encoders eagerly check that your data matches the expected schema, providing helpful error messages before you attempt to incorrectly process TBs of data

### RDD’s of JavaObjects **(vs)** Dataset’s
![image](https://user-images.githubusercontent.com/22542670/27128201-351d0b84-511b-11e7-8c08-5f0dd0b4085b.png)

### Benefits of Dataset’s
- Compact (less overhead)
- Reduce our memory footprint significantly.
- Possible in place transformations for simple one’s without the need to deserialise. (Let’s see how this happens in detail below)

### How does in-place transformation happen?
Let’s see how the same old filter fn behaves now.  Consider the case where we’ve to filter input by `year>2015` condition. Note that the filter condition specified via the data frame code `df.where(df(“year” > 2015))` is not an anonymous function. Spark exactly knows which column it needs for this task and that it needs to do greater than comparison. 
![image](https://user-images.githubusercontent.com/22542670/27127611-51e2148c-5119-11e7-93cc-7544a8e6cdf0.png)

The low-level byte code generated for this query looks something like this as shown in the above figure: 
```markdown
bool filter(Object baseObject) {
int offset = baseoffset + <..>
int value = Platform.getInt(baseObject, offset)
return value > 2015
}
```
**Interesting things to note here is that:**
- Filter fn is not anonymous to spark
- Object wasn't deserialised to find the value of column ‘year’
- The value of ‘year’ is directly fetched from the offset

### This is great!! Let’s see how does in-place transformation helps in speed?
Atleast for the simple use cases..
- We can Directly operate on serialised data => lesser Deserialisation
- Lesser Deserialisation => less Object creation 
- Less Object creation => Lesser GC 
- Hence Speed!!

### Finally: What is DataSet/DataFrame?
- Strongly typed object mapped with schema
- Stored in tungsten row-format
- At the core of the Dataset API is a new concept called an Encoder
- Encoder is responsible for converting to and from spark internal Tungsten binary format.
- Encoder uses data schema while performing any operations.
- While DataSet is a strongly typed object, DataFrame is DataSet[GenericRowObject]
- Now, that we’ve seen in-depth detail of how and why Dataset emerged and it looks so promising, Why did spark moved to column-based Parquet? Let’s find out why..

## Part2: Evolution of Data Storage Format's - from Row-based Dataset to Column-based Parquet:
As per our discussions so far, Spark 1.x used row-based storage format for Dataset’s. Spark 2.x added support for column-based format. columnar format is basically transpose of row-based storage. All of the integers are packed together, all the strings are together. Following picture illustrates the memory layout of a row-baed vs column-based storage formats.
![Image](https://user-images.githubusercontent.com/22542670/26983627-187e1384-4d5a-11e7-9856-2ae5d20071c6.png)

### Why columnar?
2nd Generation Tungsten Engine tried to use some standard optimisation techniques like loop unrolling, SIMD, prefetching etc which modern compilers and CPUs apply to speed up runtime. As part of this, Spark came up with two new optimization techniques called WholeStageCodeGeneration and Vectorization (Detail of these technologies can be found [here](https://spoddutur.github.io/spark-notes/wsg.html)). For this spark did 2 things:
1. Tweaked its execution engine to perform vector operations and attained data level parallelism at algorithm level
2. Spark moved from row-based to columnar in-memory data enabling themselves for further SIMD optimisations like data striping. (Details on this shift can be found [here](https://spoddutur.github.io/spark-notes/wsg.html))

### What opportunities did columnar format opened up?
1. **Regular data access vs Complicated off-set computation:** Data access is more regular in columnar format. For example, if we have an integer, we always access them 4 bytes apart. That’s very nice for the cpu. With row-based format there’s complicated offset computation to know where am I. 
2. **Denser storage:** Because the nature of the data is homogeneous, we can apply better compression techniques according to the data type.
3. **Compatibility and zero serialization:** Columnar format is more compatible because many high performance systems already use columnar like numpy, tensorflow etc. Add on top of it, with spark having them in memory implies zero serialisation and zero copy. For example, most of our spark plan is evaluated in spark and at the end of it we want to call tensor flow and when its done, we want to get back. With spark using columnar in-memory format, that’s compatible with tensorflow. So, its gonna be done without ever having to do serialisation etc. It just works together. 
4. **More Extensions:** Lastly, having columnar format gives us more options to extend the system in future. When the data is homogeneous, its easier to process encoded data in place. It also improves spark’s in-memory columnar-cache. With Columnar + Vectorization, we can leverage GPU’s better i.e., you can offload to GPU better and avail its advanced hardware techniques.

**Performance Benchmarking:** Parquet which is the most popular columnar-format for hadoop stack was considered. Parquet scan performance in spark 1.6 ran at the rate of 11million/sec. Parquet vectored is basically directly scanning the data and materialising it in the vectorized way. Parquet vectorized ran at about 90 million rows/sec roughly 9x faster. This is promising and clearly shows that this is right thing to do.
![Image](https://user-images.githubusercontent.com/22542670/26983657-2e3aaa84-4d5a-11e7-8848-fd84d6ee0612.png)
### CONCLUSION - Journey of Data Layout in Spark

**We’ve seen:**
- Disadvantages with java objects
- Evolution of strongly typed Datasets with schema using the new internal tungsten row-format
- Dataset Benefits: lesser serialisation/deserialisation, less GC, less object creation and possible In-place transformations for simpler cases.
- Spark introduced support for columnar data using a new technique called Vectorization in spark 2.x to better leverage the advancements of Modern CPU’s and hardware like loop-unrolling, SIMD etc.

## References
- [Spark Memory Management](https://www.youtube.com/watch?v=dPHrykZL8Cg)
- [Deep Dive into Project Tungsten](https://www.youtube.com/watch?v=5ajs8EIPWGI)


