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



there are many compaction algorithm available (such as [zstd](https://github.com/facebook/zstd)) 对zstd 的发明人感兴趣的可以听[这一期 podcast](https://corecursive.com/data-compression-yann-collet/#), 还是很有意思的




