[[Chapter 2]] is written in programmers perspective about data model
This chapter will discuss the same from database's point of view: *how* we can store the data that we're given, and *how* we can find it again when we're asked for it

First, why we care how the database handles storage and retrieval internally? 

Although we don't need to write our own storage engine, we *do* need to choose a storage engine that is appropriate for our application 

Two families of storage engines are discussed: *log-structured* storage engines, and *page-oriented* storage engines such as B-trees

## Data structures that power your database
simplest kv pair database
```bash
#!/bin/bash
db_set () {
	echo "$1,$2" >> database
}
db_get () {
	grep "^$1," database | sed -e "s/^$1,//" | tail -n 1
}
```

`db_set` has actually really good performance $O(1)$ because it is append only at the end of the file. But `db_get` has $O(n)$ because it needs to look up the entire database for a key 

绝大多数 database 以及 OS 都有一个 append only log (也叫 write ahead log), 就是类似于这个 append only file. 只不过 DBMS 或者 OS 要保证定期清理这个log 以避免不断增长。distributed system 其实也是基于一个log 来的，感兴趣请看[the log](https://engineering.linkedin.com/distributed-systems/log-what-every-software-engineer-should-know-about-real-time-datas-unifying) 这篇文章

>In order to efficiently find the value for a particular key in the database, we need a different data structure: an *index*.

这里就要引出 index 的概念了。This chapter will look at a *range* of indexing structures and how they compare 

> The general idea behind them is to keep some additional metadata on the side

就是通过 meta data (pointer etc) 来帮助 locate the data 

> An index is an *additional* structure that is derived from the primary data

索引是由主要数据衍生出来的数据, 就像每本书后面的索引一样，用来帮助 locate data 
维护一个additional structure 就会有 overhead，especially on writes because index needs to be updated every time data is written

所以 database default 是不会给所有数据做 index 的，而是 application developer 来选择哪些数据需要 index 
### Hash Indexes
Hash index 估计是最常见的 index 形式了，java 的 `Map`, python 的 `dict` 以及 json 都是key-value pair 

既然已经有 in memory data structure 了，on disk 也可以用 hash 来 index 
加入我们有一个 append only file, 最简单的一个 indexing strategy 就是在 in memory hash map 里面存一个你要查找的 key， value 则是 byte offset for the append only file, 想要查哪个值，直接去对应的file 的 offset里面读取就可以了

![[hash_index_disk.png]]
>This may sound simplistic, but it is a viable approach. In fact, this is essentially what Bitcask (the default storage engine in Riak) does

想算一个url 的点击数量这种 workload 就很适合 Bitcask 这种 storage engine (a lot of writes, not too many distinct keys (only urls))
in other words, large number of writes per key, and keys are small enough to fit in memory

#### How to avoid running out of disk space? 
break the log into segments and when a segments reach to a certain size (lets say 5mb) making subsequent write to a new segment file and perform *compaction* on these segments. 
![[compaction_example.png]]
这里的compaction 就直接把最新的值记录下来，而且可以在 background thread 进行
>while it is going on, we can still continue to serve read and write requests as normal, using the old segment files. After the merging process is complete, we switch read requests to using the new merged segment instead of the old segments—and then the old segment files can simply be deleted.

![[compaction_and_merge.png]]
每一个 segment 在 memory里面都有自己的 hash table, mapping keys to file offset. In order to find value for a key, we first check the most recent segment hash map; if the key is not present we check the second most and so on
```java
Map<key, data> seg1;
Map<key, data> seg2; # <- check this first 
...
```
The merging process keeps the number of segments small 

There is a lot of details in implementation:

*File format*
CSV is not best format. binary is faster and simpler
*Deleting records*
Append a special deletion record to the data file (sometime called a *tombstone*) when log segments are merged, tombstone tells the merging process to ignore any previous value for deleted key
*Crash recovery*
If database is restarted, in memory hash maps are lost. In principle you can restore it by reading the entire segment file. but it takes too long if segment file gets large. Bitcask speeds up recovery by storing snapshot of each segment's hashmap on disk 
*Partially written records*
database may crash at any time, including halfway through appending to the log. Bitcask files include checksum, allowing such corrupted parts to be detected and ignored
*Concurrency control*
Since it is append only, a common implementation is to have 1 writer thread. And many reader threads.

Compare to update file in place, append only design has following advantages:
- *Sequential writes are generally much faster* than random writes. Especially on spinning disk. To some extent sequential writes are preferable in SSD [4](https://www.cs.purdue.edu/homes/csjgwang/pubs/TOIS14_SSDWebCache.pdf)
- *Concurrency and crash recovery are much simpler* if segment files are appendonly or immutable. For example, you don’t have to worry about the case where a crash happened while a value was being overwritten, leaving you with a file containing part of the old and part of the new value spliced together.
- Merging old segments avoids the problem of data files getting fragmented over time.

But it also has limitations:
- must fit in memory
- range queries are not efficient

### SSTables and LSM-Trees
SSTable 就是在之前 append only segments的基础上让 key sorted and unique。所以叫做 *Sorted String Table* (SSTable for short) 
SSTables 相比于 hash indexed log segment 有几个好处
1. *Merging segments is simple and efficient* (similar to mergesort)
![[SSTable_Merge_example.png]]
if one key appears more than 1 segment, keep the most recent one
2. *No longer need to keep index of all the keys in memory*. 
Just like binary search tree, you could get the floor key of current key your are looking for. Then get into the segment you need. 
$handb < handi < hands$
![[SSTable_sparse_in_memeory_index.png]]
This way the index is sparse which allow us to store more keys (each segment can be few kilobytes so that it can be scanned quickly). Just like page fault in memory, it will also go to disk to bring relevant file content in memory
3. Compress a segment into a block before write to disk and save I/O

#### Constructing and maintaining SSTables
How do you get your data to be sorted by key?
In memory we have well known data structure 
- BST
- red black tree 
- AVL trees
We can now make our storage engine work as follows:
- When write comes in, add it to our in memory balanced tree data structure (red black tree). Those in memory tree is sometimes called *memtable*
- when the in memory tree gets bigger than some threshold (few megabytes), write it out to disk as an SSTable file. This file will become the most recent *segment* of the database. While SSTable is being written out to disk, writes can continue to a new memtable instance
- In order to serve a read request, first try in memory tree. If key is not found, then most recent on-disk segment, then older etc.
- From time to time, run merge and compaction process in the background to combine segment files and discard the overwritten and deleted value

This technique works well except when database crashes. The most recent writes are lost (in memory). We can keep a separate log on disk to avoid this problem. This log is append only and does not require key to be sorted because it is for recovery only. We could discard the log once the in memory tree is written to disk

#### Making an LSM-tree out of SSTables
LevelDB and RocksDB essentially use the technique above to build their storage engine. 
key-value engine libraries that are design to be embedded into other applications. 
LevelDB can be used in Riak as an alternative to Bitcask. Similar storage engine are used in Cassandra and HBase [8](https://blog.cloudera.com/apache-hbase-i-o-hfile/), which are inspired by [BigTable](https://research.google/pubs/pub27898/) 

*Log-Structured merge tree* (LSM tree) originated from [this paper](https://www.cs.umb.edu/~poneil/lsmtree.pdf). Storage engines based on this principle of merging and compacting sorted files are called *LSM* storage engines.

Lucene(indexing engine for Elaticsearch and Solr) uses a similar method for storing its term dictionary, where key is the *term* (word) and the value is the list of IDs of all the documents (posting list) that contain the word 

In Lucene, this this mapping from term to posting list is kept in SSTable-like sorted files

#### Performance optimizations
A lot of details when implementation happens. LSM-tree algorithm can be slow when looking up keys that do not exist. (check in memory tree, check all segment files etc) 

In order to optimize this, storage engine often use [*Bloom filters*](https://people.cs.umass.edu/~emery/classes/cmpsci691st/readings/Misc/p422-bloom.pdf), which tells you if a key does not appear in the database, and thus saves many unnecessary disk reads for nonexistent keys.

There are also different strategies of how SSTables are compact and merged.
>The most common options are *size-tiered* and *leveled compaction*.

In sized-tiered compaction, newer and smaller SSTables are successively merged into older and larger SSTables. 

In leveled compaction, the key range is split up into smaller SSTables and older data is moved into separate "levels", which allows the compaction to proceed more incrementally and use less disk space.

>the basic idea of LSM-trees—keeping a cascade of SSTables that are merged in the background—is simple and effective. Even when the dataset is much bigger than the available memory it continues to work well. Since data is stored in sorted order, you can efficiently perform range queries (scanning all keys above some minimum and up to some maximum), and because the disk writes are sequential the LSM-tree can support remarkably high write throughput.


## B-Trees



there are many compaction algorithm available (such as [zstd](https://github.com/facebook/zstd)) 对zstd 的发明人感兴趣的可以听[这一期 podcast](https://corecursive.com/data-compression-yann-collet/#), 还是很有意思的




