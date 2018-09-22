# Why one should avoid filesystem as source in spark?

I was experimenting to compare the usage of filesystem (like s3, hdfs) VS a queue (like kafka or kinesis) as data source of spark i.e.,
```markdown
//1. filesystem as datasource
spark.read.json(“s3://…<S3_PATH>..”) (VS)

// 2. kinesis as datasource
spark.read.format(kinesis)
```
As I started analysing s3 as source, at first it looked all dandy - easy of use, reliable uptime, easy maintenance etc. On top of that, its checkpointing system also seemed fine. Checkpointing essentially keeps track of the files it finished processing or partly processed where in spark maintains a list of files it processed and also the offsets of the files that are partly processed. So, the next time our spark application kicks-off, it’ll not reprocess everything in s3 bucket. Instead, spark will pick up the last partially processed file according to the saved offsets in checkpoint and continue from there.

#### Trouble started once I deployed. 
#### Culprit: What files to pick for next batch?

## How does spark decide next batch of files to process?
Per every batch, it repeatedly lists all of the files in s3 bucket in order to decide the next batch of files to process as shown below:
- **Compute all files in s3 bucket** - Spark calls the list API to list all the files in s3 
- **Compute processed files in previous runs** - It then pulls the list of files it processed from checkpoint
- **Compute files to process next = Subtract(AllFiles - ProcessedFiles)**

Listing api of S3 to get all files in the bucket is very expensive. Some numbers to show how slow it is:
- **80-100 files in s3 bucket takes ~2-3secs time to list**
- **>500-1000 files in s3 bucket takes ~10secs time**
- **>1000-10000 files in s3 bucket takes ~15-20secs time**

#### Note that the numbers didn’t change much with HDFS file system as source for Spark

## What does this mean?
If we use a file system as spark-source, supposing that our batch interval is say 30secs, ~50% of the batch-processing time is taken just to decide the next batch to process. This is bad. 
**On top of that, this listing happens repeated per every batch - absolute waste of resources and time!!!**

Hence, S3 as source for Spark is bad for following reasons: 
1. **High Latency:** Need to list large buckets on S3 per every batch, which is slow and resource intensive.
2. **Higher costs:** LIST API requests made to S3 are costly involving network I/O.
3. **Eventual Consistency:** There is always a lag observed in listing the files written to S3 because of its eventual consistency policy.

## Solutions:
Before we jump into solution, I want you to make note of the fact that S3 is not a real file system i.e., the semantics of the S3 file system are not that of a POSIX file system. So s3 file system may not behave entirely as expected. This is why it has limitations such as eventual consistency in listing the files written to it. 

### Idea: 
- The core idea of the solutions mentioned below is to `avoid list api in S3` to compute the next batch of records to process.
- For this, one should establish alternative secondary source to track files written to S3 bucket.

### Netflix's solution:
Netflix has addressed this issue with a tool named [S3mper](https://github.com/Netflix/s3mper). It is an open-source library that provides an additional layer of consistency checking on top of S3 through the use of a consistent, secondary index. Essentially, any files written to S3 are tracked in the secondary index and hence makes listing files written to S3 consistent and fast.

After looking at Netflix’s solution, it is now clear that, if we want to avoid repeated listing of files in S3 bucket, we need an alternative secondary source to track the files in S3 bucket.

### DataBrick's solution:
DataBricks has implemented a solution to address the same but its unfortunately not open-sourced.  Instead of maintaining a secondary index, they used SQS queue as secondary source to track the files in S3 bucket.
- So, every time a new file is  written to S3 bucket, its path is added to SQS queue.
- This way, in our spark application, to find out the next files to process, we just need to poll SQS.
- Processed files will be removed from the queue because polling removes those entries from SQS.
- Therefore, we need not call list api in S3 anymore to find out the next batch records.

### Conclusion and key takeouts:
- For a streaming application, Queues should be the first choice of streaming-sources for spark applications because here we just poll the queues to get the next batch of records. More importantly neither latency nor consistency is a problem here.
- Any filesystem, be it S3 or HDFS, comes with the payloads of eventual consistency or/and high latency. Such filesystems as sources are fine to experiment a POC quickly as it wouldn't need any setup process like Kafka Queue.
- But for production purposes, if we intend to use filesystem, then we definitely need a solution like [Netflix’s S3Mper](https://github.com/Netflix/s3mper) or [Databrick’s S3-SQS](https://docs.databricks.com/spark/latest/structured-streaming/sqs.html) where we can get rid of S3 List API call. For this:
- One should implement a custom solution such as secondary index or a queue which holds next batch of files written in S3 bucket.
- [Register our custom solution as DataSource with Spark](https://jaceklaskowski.gitbooks.io/mastering-spark-sql/spark-sql-DataSourceRegister.html)

I’ll try to provide a sample custom implementation for the same soon and share here..
