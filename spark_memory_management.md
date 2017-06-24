## Spark Memory Management
In this blog, we'll address following 2 questions:-
- **What are the memory needs of a task?**
- **How does Spark arbitrate memory between Execution and Storage?**
- **How is memory shared among different tasks running on the same worker node ?**

### 1.What are the memory needs of a task?
Task is basically smallest unit of execution that represents a partition in our dataset. Every task needs 2 kinds of memory: 
1. **Execution Memory:** 
  - Execution Memory is the memory used to buffer Intermediate results.
  - As soon as we are done with the operation, we can go ahead and release it. Its short lived.
  - For example, a task performing Sort operation, would need some sort of collection to store the Intermediate sorted values.
2. **Storage Memory:** 
  - Storage memory is more about reusing the data for future computation. 
  - This is where we store cached data and its long-lived. 
  - Until the allotted storage gets filled, Storage memory stays in place. 
  - LRU eviction is used to spill the storage data when it gets filled.

**`Let’s illustrate it with an example task of "Sorting a collection of Int’s"`**

![image](https://user-images.githubusercontent.com/22542670/27504472-8269bc82-58a7-11e7-9a40-7e3900055a3f.png)

### 2.How does Spark arbitrate memory between Execution and Storage?
Simplest Solution – **Static Assignment**
- Static Assignment approach basically splits the total available on-heap memory (size of your JVM) into 2 parts one for ExecutionMemory and the other for StorageMemory. 
- As the name says, this memory split is static and doesn't change dynamically. 
- This has been the solution since spark 1.0 (May 2014). 
- While running our task, if the execution memory gets filled, it’ll get spilled to disk as shown below:
![image](https://user-images.githubusercontent.com/22542670/27504478-dae23b28-58a7-11e7-9750-4aca0a6203a6.png)
- Likewise, if the Storage memory gets filled, its evicted via LRU
![image](https://user-images.githubusercontent.com/22542670/27504731-67d537ec-58ad-11e7-8f61-f24b3dae9f99.png)

**Disadvantage:** Even if the task doesn't have any StorageMemory need, the ExecutionMemory will not be able to use all of the available memory
![image](https://user-images.githubusercontent.com/22542670/27504510-8e3ee72a-58a8-11e7-879b-3d615bf9b8ab.png)

**How to fix this?**

`UNIFIED MEMORY MANAGEMENT` - This is how Unified Memory Management works:
- Express execution and storage memory as one single unified region (i.e., on-heap memory is not split. Its shared between execution and storage memory combinedly)
- Keep acquiring execution memory and evict storage as u need more execution memory. 

Following picture depicts Unified memory management..

![image](https://user-images.githubusercontent.com/22542670/27504536-2e56d1c8-58a9-11e7-9a51-d8b7120c651a.png)

**But, why to evict storage than execution memory?**

Spilled execution data is always going to be read back from disk vs cached data may or may not. (User might tend to aggressively cache data at times with/without its need.. )

**What if application relies on caching like a Machine Learning application?**
We cant just blow away cached data like that in this case. So, for this usecase, spark allows user to specify minimal unevictable amount of storage a.k.a cache data. Notice this is not a reservation meaning, we don’t pre-allocate a chunk of storage for cache data such that execution cannot borrow from it. Rather, only when there’s cached data this value comes into effect..

### 3.How is memory shared among different tasks running on the same worker node?
Ans: **Static Assignment (again!!)** - No matter how many tasks are currently running, if the worker machine has 4 cores, we’ll have 4 fixed slots.
![image](https://user-images.githubusercontent.com/22542670/27504541-465957aa-58a9-11e7-9626-9ad4f6b077a3.png)

**Drawback:** Even if there’s only 1 task running, its gonna get only one-quarter of the total memory. 

### Better Solution – Dynamic Assignment (Again)!!
More efficient alternative is Dynamic allocation where how much memory a task gets is dependent on total number of tasks running. If am the only task running, I can feel free to acquire all the available memory.

![image](https://user-images.githubusercontent.com/22542670/27504542-4922ffa4-58a9-11e7-97ff-d10d2d749611.png)

As soon as another task comes in, we’ll have to spill to disk and free space for task2 for fairness. So, number of slots are determined dynamically depending on active running tasks.
![image](https://user-images.githubusercontent.com/22542670/27504544-4cae9f8e-58a9-11e7-9d6a-adc90fe66dec.png)

**Key Advantage:**
One notable behaviour here is that if we have a straggler which is a last remaining task.. these are potentially expensive because everybody is already done but then u are the last remaining one. This model allocates all the memory to the straggler because number of actively running tasks is one. 
This has been there since spark 1.0 and its been working fine since then. So, Spark haven't found a reason to change it.
![image](https://user-images.githubusercontent.com/22542670/27504547-521bd842-58a9-11e7-96ad-4c08f351f72d.png)
## CONCLUSION - MEMORY Management
`We understood:`
- Two kinds of memory needs per task
- How to arbitrate within a task (i.e., between execution and storage memory of a single task)
- How to arbitrate memory between multiple tasks
- **Common Solution:** Instead of statically reserving memory, force memory to spill when there’s memory contention. So, essentially, solve memory contention lazily rather than eagerly. 
- Static assignment is simpler
- Dynamic allocation handles stragglers better
