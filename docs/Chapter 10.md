A system cannot be successful if it is too strongly influenced by a single person. Once the initial design is complete and fairly robust, the real test begins as people with many different viewpoints undertake their own experiments.
	—Donald Knuth

Web made request/response style of interaction so common where people forget there are other ways to interact with the system. There are 3 types of systems:
*Services* (online systems)
	A service waits for a request from a client to arrive. When it receive a request, it tries to handle it as soon as possible and sends a response back. 
*Batch processing systems* (offline systems)
	A batch processing system takes a large amount of input data, runs a *job* to process it, and produces some output data. (few minutes to several days)
*Stream processing systems* (near-real-time systems)
	Somewhere between online and offline/batch processing. Like batch processing, stream takes input and produces output, but stream job operates on events. 

MapReduce provides a clear picture of why and how batch processing is useful. Batch processing is very old form of computing. Punch card tabulating machines implemented a semi-mechanized form of batch processing to compute aggregate statistics from large inputs. 

## Batch Processing with Unix Tools
Main idea of unix tools is that it is able to pipe current program's output into next program's input. This way we can start simple log analysis
#### Simple log analysis
download sample log from NASA by
```bash
curl "ftp://ita.ee.lbl.gov/traces/NASA_access_log_Jul95.gz" > NASA.gz
gunzip NASA.gz
```
below is the command made from unix tools to perform count on ip address on NASA's webserver log
```bash
cat NASA | awk '{print $2}' | sort | uniq -c | sort -r -n | head -n 5
```
sample output
```
17572 piweba3y.prodigy.com
11591 piweba4y.prodigy.com
9868 piweba1y.prodigy.com
7852 alyssa.prodigy.com
7573 siltb10.orl.mmc.com
```

This case you just finished a data pipeline from unix tool

Similar logic can be applied by using a programing language
```ruby
counts = Hash.new(0)

File.open('./NASA') do |file| 
	file.each do |line|
		url = line.split[6]
		counts[url] += 1 
	end
end
top5 = counts.map{|url, count| [count, url] }.sort.reverse[0...5]
top5.each{|count, url| puts "#{count} #{url}" }
```

#### Sorting versus in-memory aggregation
Although the logic is similar in both way of processing log. Main difference is execution flow. If your working set (size of your data), is small to fit in memory, then programming language way might be sufficient because all data can be stored in memory. However, if your working set is larger than memory available (let's say 1TB log file need to be processed). Then unix tool approach is better. Because sort utility in GNU Coreutils (Linux) automatically handles larger-than- memory datasets by spilling to disk, and automatically parallelizes sorting across multiple CPU cores.

Martin spent lot of effort explaining unix philosophy which simply comes down to every program should do one thing well. And Unix use uniform interface, i.e. **everything is a file** to decouple program and it's input/output. This way program can be composed together in a way their designer could never think of. 

This philosophy is very similar to Agile/Devops practice today


## MapReduce and Distributed Filesystems
MapReduce is just like unix tools that build for processing data. 
Main two function you need to implement is *mapper* and *reducer*. Where mapper is like `awk '{print $2}'` where it extract from input and map it to a key-value pair. `sort` or `uniq -c` will be your *reducer* where it process the input and output to somewhere. 

Main difference between MapReduce and unix tool is MapReduce job can be deployed to multiple machines where unix tool usually work on a single machine. 

![[map_reduce_job_ex.png]]

Chained MapReduce jobs are therefore less like pipelines of Unix commands (which pass the output of one process as input to another process directly, using only a small in-memory buffer)

Chained MapReduce jobs can sometimes hard to read so higher level tools have been developed to help with that 
Such as Hadoop, such as Pig [30], Hive [31], Cascading [32], Crunch [33], and FlumeJava [34], also set up workflows of multiple MapReduce stages that are automatically wired together appropriately.

### Reduce Side Joins and Grouping
This section goes deeper into how `JOIN` operator was implemented. In most dataset, data have relation to each other. A *foreign key* in relational model, a *document reference* in document model, or *edge* in a graph model 

Those can be used as index to look stuff up then performing joins. When MapReduce job given set of files as input, it will scan the entire file and this is called *full table scan* in database terms. 

#### Sort merge joins
Mapper is basically to extract a key and value from each input record. 
![[map_reduce_ex.png]]
when MapReduce partitions the mapper output by key and then sorts the kv pairs, the effect is that all the activity events and the user record with same user ID become adjacent to each other in reducer input 

after sorting, join logic become easy: the reducer function is called once for every user ID, and first value will be dob from user db. Reducer iterates over activity events with the same user ID, outputting `viewed-url` and `viewer-age-in-years` pair 

Subsequent MapReduce job can calculate the distribution of viewer ages for each URL

One way of looking at this architecture is mapper sends messages to reducers 

#### GROUP BY
Besides joins, another common use of bring related data to same place is grouping records by some key (`GROUP BY` clause in SQL) and perform some kind of aggregation 
- counting
- adding up particular field
- pick top k record

Another common use for grouping is *sessionization* where such analysis could be used to work out whether users who were shown a new version of your website are more likely to make a purchase than those who used old version (A/B testing)

*hot keys* are those celebrities on social network website. Collecting all activity related to a celebrity can lead to skew (aka *hot spots*) that is, one reducer are processing significantly more records than the others 

There are method like sampling to determine which keys are hot. Other approach like sharded join (predetermined hot spot). 
### Map Side Joins
Previous section talked about join logic on reducers. You don't need to make any assumption about the input data on reducer side join. However, the downside is that all that sorting, copying to reducers, and merging of reducer inputs can be expensive. 

On the other hand, if you can determine your input data format, it is possible to make joins faster by using map side join. Each mapper read from distributed file system and output to file system
#### Broadcast hash joins
the word broadcast reflects the fact that each mapper for a partition of the large input reads the entirety of the small input (so small input is "broadcast" to all partitions of the large input) and this small dataset is loaded into memory 

#### Partitioned hash joins
partition and map to same way. For example, arrange activity events and user database to each be partitioned based on last decimal digit of user ID. (10 partition in total). Mapper. 3 first loads all users with ID ending in 3 into hash table, then scans over all activity events for each user whose ID ends in 3

#### Map side merge joins
On top of partitioned in the same way, we can also sort the key. In this case, it does not matter whether the inputs are small enough to fit in memory, because mapper can perform same merging operation that would be done by reducer. 

This usually mean there is previous map reduce job that partition and sort the key already. 

### The Output of Batch Workflows
Compare to OLTP where generally look up small number of records by key, analysis workload scan over a large number of records. Performing groupings and aggregations and output is often some form of report: a graph showing the change in a metric over time, top 10 items according to some ranking. The consumer of such a report is often an analyst or manager who make business decisions. Below are different use cases 

#### Building search indexes
MapReduce was used for building index for search engine in Google. If we need to perform full text search over a fixed set of documents, then batch process is a very effective way of building indexes: mappers partition the set of document as needed. Each reducer builds index for its partition, and index file are written to distributed file system

#### KV stores as batch process output
Another common output from batch processing is to build machine learning systems such as classifiers (e.g. spam filters, anomaly detection, image recognition) and recommendation systems (e.g. people you may know, products you may be interested in, or related searches)

The output of those jobs are some kind of database: for example, a database that can be queried by user ID to obtain suggested friends for that user. 

These database can be files from batch jobs and read only. Since mapper usually sort key and map them together

#### Philosophy of batch process
Since input is left unchanged, we can experiment with it very easily. (no side effects)
- If you introduce a bug into the code and the output is wrong or corrupted, you can simply roll back to a previous version of the code and rerun the job, and the output will be correct again.
- As a consequence of this ease of rolling back, feature development can proceed more quickly than in an environment where mistakes could mean irreversible damage.
- If a map or reduce task fails, the MapReduce framework automatically re- schedules it and runs it again on the same input.
- Like Unix tools, MapReduce jobs separate logic from wiring (configuring the input and output directories), which provides a separation of concerns and ena‐ bles potential reuse of code

### Comparing Hadoop to Distributed Databases
Hadoop is like a distributed version of Unix tools. Massively parallel processing databases have been explored this idea previously. (such as Teradata, NonStop SQL) 

The main difference is MPP databases focus on parallel execution f analytic SQL queries where combination of MapReduce and distributed filesystem provide something more like a general purpose OS that can run arbitrary programs 

Database requires you to define schema but filesystem does not. they can equally well be text, images, videos, sensor readings, sparse matrices, feature vectors, genome sequences, or any other kind of data.

Hadoop enable dumping data into HDFS and figuring out later how to process it. While MPP databases need careful up-front modeling

The idea is similar to data warehouse. And this data dumping shifts the burden of interpreting the data from producer to consumer (the interpretation of the data)

This can be advantage if producer and consumer of the data are in different teams. 

Hadoop has been used for ETL process. Data is dumped into distributed filesystem in raw form. MapReduce jobs transform it into a relational form and import it into an MPP data warehouse for analytic purposes. 
## Beyond MapReduce


## Summary

