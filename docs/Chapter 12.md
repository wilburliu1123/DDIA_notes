This chapter will look into the future of data systems
## Data Integration 
For complex applications, data often used in several different ways. We often have to use multiple kind of database together to accomplish one goal

### Combining Specialized Tools by Deriving Data
It is common to combine OLTP database with full-text search index. This can be done by introducing log based message broker that has CDC feature implemented. We might arrange data to write to a system of record database and use CDC to forward those change to search index in the same order. 

Allowing application directly write to both database and search index will introduce race condition. 

#### Derived data vs distributed transactions 
Classic approach for keep different data system is to use distributed transactions such as 2PC. At an abstract level, 2PC using lock for mutual exclusion while CDC and event sourcing use a log for ordering. Distributed transactions use atomic commit to ensure that changes only effect exactly once, while log-based systems are based on deterministic retry and idempotence. 

Biggest difference is transaction system provide linearizability where derived data system often updated asynchronously, so they do not offer same timing guarantees by default

Because of lack support of good distributed transaction protocol, Martin believe that log based derived data is the most promising approach for integrating different data systems. 

#### The limits of total ordering
When system scaled bigger enough, limitations of total ordering begins to emerge:
- In most cases, constructing a totally ordered log requires all events to pass through a *single leader* that decides on the ordering. If the throughput of events is greater than single machine can handle, we need to partition it across multiple machines. The order across different partitions is ambiguous
- If servers are spread across multiple geographically distributed datacenters, where you would like to tolerate entire data center go offline. This typically requires a separate leader in each datacenter. This implies undefined ordering of events that originate in two different datacenters.
- For *microservices*, a common design is to deploy storage unit as an independent unit. So no two services share same storage system. When two events originate in different services, there is no defined order for those events. 
- Some application maintain client side state where user input get immediately updated and continue to work offline. With such application, clients and servers are very likely to see events in different orders.

> In formal terms, deciding on a total order of events is known as *total order broadcast*, which is equivalent to consensus. 

Most consensus algorithms are designed with single node throughput. It is still an open research problem to design consensus algorithms that can scale beyond the throughput of a single node

#### Ordering events to capture causality
In social network app, after a couple break up, they should not see each other's rude message. This has a *unfriend* event followed by *message* event. If this causal dependency is not captured, message will send notification to the user. Starting point to solve this problem could be 

- Logical timestamp can provide total ordering so they can help and require recipient to handle events that is out of order.
- Conflict resolution algorithm 

### Batch and Stream Processing 
There are also many detailed differences in the ways the processing engines are implemented, but these distinctions are beginning to blur.

Spark performs stream processing on top of a batch processing engine by breaking the stream into microbatches, whereas Apache Flink performs batch processing on top of a stream processing engine [5].

#### Maintaining derived state
Batch process has a strong functional flavor where outputs are deterministic and can be retried many times. No matter what derived data is (cache, search index, or statistical model), it can be think of as a data pipeline. In principle derived data should be maintain synchronously but asynchronous processing make the system more robust (log based message system)

#### Reprocessing data for application evolution
Schema migration on railway is a very interesting example
Similarily, batch and stream processing can be used for schema evolution. 
>You can then start shifting a small number of users to the new view in order to test its performance and find any bugs, while most users con‐ tinue to be routed to the old view. Gradually, you can increase the proportion of users accessing the new view, and eventually you can drop the old view [10].

#### Lambda architecture
*lambda architecture* is the proposal to combine batch and stream processing. 
>The lambda architecture proposes running two different systems in parallel: a batch processing system such as Hadoop MapReduce, and a separate stream- processing system such as Storm.

Although lambda architecture was an influential idea, Martin think it has couple of practical problem:

- Maintaining batch and stream process at the same time is an additional effort. 
- Stream pipeline and batch pipeline produce separate output and it can be hard to join them together 
- for extreme large dataset, batch pipeline need to setup incremental batch which can have timing issues

#### Unifying batch and stream process
This can give multiple benefits:
- Replay historical events from same stream engine
- Exact once processing semantics 
- Tools for windowing by event time

## Unbundling Databases
At most abstract level, databases, Hadoop, operating systems are all store some data and allow you to process and query those data. 

Database store data in row and column where OS store it in file, but at its core both system are information management system

Unix and relational databases have approached information management problem with different philosophies. Unix would like to provide programmer an abstract view of low level hardware. Whereas relational databases give application programmer a high level abstraction that hides complexities of data structures on disk, concurrency and crash recovery. 

Unix is basically thin wrapper around hardware resources and relational databases can draw a lot of powerful infrastructure (query optimization, indexes, join methods, concurrency control, replication etc) 

Both Unix and relational model emerged in 1970s and those different approach still effect today's world. Martin would interpret NoSQL movement as apply a Unix approach of low level abstraction in distributed OLTP domain 

Martin will try to reconcile the two philosophies in this section and hope to combine both worlds 

### Composing a Data Storage Technologies 
Over the course of the book, we have seen different feature provided by databases
- Secondary indexes 
- materialized views (precomputed cache)
- Replication logs 
- Full text search indexes 

Those features are very similar that batch and stream process are trying to perform
#### Creating an index
When `CREATE INDEX` run on relational database, it will scan over a consistent snapshot of the table, sort them, and write it out. This process is very similar setting up a new follower or bootstrapping change data capture in stream system

#### The meta database of everything
The dataflow across entire organization starts looking like one huge database. Whenever, batch, stream, or ETL process transport data from one place to another, it is like keeping this huge database subsystem up to date. 

Martin think there are two ways to compose different databases into a cohesive system

*Federated Databases*: Unifying reads
It is possible to provide unified query interface to different storage engine and processing methods. This approach is called *federated database* or *polystore* 
> Applications that need a specialized data model or query interface can still access the underlying storage engines directly, while users who want to combine data from disparate places can do so easily through the federated interface.

JDB can use specialized data structure to speed up read?

*Unbundled databases*: Unifying writes 
Through change data capture or event logs, we could synchronized write across different technologies. This approach is like Unix tradition of small tools that do one thing well and communicate through uniform API (pipes) and composed using a higher level language (shell)

#### Making unbundling work
Martin believe that an asynchronous event log with idempotent writes is a much more robust and practical approach.

The big advantage of log-based integration is *loose coupling* between the various components 
1. Asynchronous event stream make system more robust against individual components failure. 
2. Unbundling data systems allows each component developed, maintained and independently from different teams. Thus allow each team focus on doing one thing well. 

#### Unbundled vs integrated system
>The goal of unbundling is not to compete with individual databases on performance for particular workloads; the goal is to allow you to combine several different data‐ bases in order to achieve good performance for a much wider range of workloads than is possible with a single piece of software. It’s about breadth, not depth

### Designing Applications Around Dataflow
Composing specialized storage and process systems with application code is called database inside out approach ([Martin's talk on this](http://martin.kleppmann.com/2015/03/04/turning-the-database-inside-out.html)) 

The term unbundling in this context was proposed by Jay Kreps [7].

In this section Martin will expand on these ideas and explore some ways of building applications around the ideas of unbundled databases and dataflow.

#### Separation of application code and state
~~Deployment and cluster management tools such as Mesos, YARN, Docker, Kubernetes, and others are designed specifically for the purpose of running application code. 
~~
In a typical web application, database is like a shared mutable variable that can be accessed synchronously over the network

>However, in most programming languages you cannot subscribe to changes in a mutable variable—you can only read it periodically. Unlike in a spreadsheet, readers of the variable don’t get notified if the value of the variable changes. (You can implement such notifications in your own code—this is known as the observer pattern— but most languages do not have this pattern as a built-in feature.)

#### Dataflow: Interplay between state changes and application code 
Thinking about applications in terms of dataflow allows us to rethink relationship between application code and state management.

#### Stream processors and services 
SOA (service oriented architecture) allow organizational scalability by decoupling. 

>Composing stream operators into dataflow systems has a lot of similar characteristics to the microservices approach [40].

However, underlying mechanisms are different. microservices require synchronous request/response interaction where dataflow system is one direction and asynchronous 

Dataflow system sometimes can achieve better performance. For example, when a customer purchase a product at one currency but pay it in another currency, there are two approaches to this problem: 
1. In microservices approach, purchase could query an exchange rate service in order to obtain the current rate for particular currency
2. In dataflow approach, code that processes purchases would subscribe to stream of exchange rate and recorded it in local database. So when it processing purchase request, it only need to query local database.

Second approach will not only be faster but more robust to the failure of another service. 

### Observing Derived State
![[write_read_path.png]]
write path is where data is stored into the system and read path is when user request for a value. 
Derived data is where write path meet read path

#### Materialized views and caching
Full text search is a good example: write path updates the index and read path searches the index for keywords. 

>If you didn’t have an index, a search query would have to scan over all documents (like grep), which would get very expensive if you had a large number of documents. No index means less work on the write path (no index to update), but a lot more work on the read path.

> On the other hand, you could imagine precomputing the search results for all possible queries. In that case, you would have less work to do on the read path: no Boolean logic, just find the results for your query and return them. However, the write path would be a lot more expensive

>Another option would be to precompute the search results for only a fixed set of the most common queries. This would generally be called a *cache* of common queries, although we could also call it a materialized view, as it would need to be updated when new documents appear that should be included in the results of one of the common queries.

#### Stateful offline capable clients
The client/server model in which clients are largely stateless and servers have the authority of the data is so common where we forget the other possibility

Recent single page application allow client side user interface and local storage in the web browser. Mobile apps can also store a lot of state on device and don't need network round-trip 

These capabilities led to renewed interest in offline-first applications that process as much as request locally as possible and sync with server in the background.

When we move away the assumption of stateless client, a world of new opportunities opens up. we can think of on-device is a *cache state* of the server. The model objects are a local replica of state in a remote datacenter.

#### Pushing state changes to clients 
In a typical web page, server don't do anything until you submit a request or reload the page. 

More recent protocol such as server-sent events and WebSocket provide communication channel where server maintain an open TCP connection with browser. This allows server actively push changes to client to prevent stale cache

>In terms of our model of write path and read path, actively pushing state changes all the way to client devices means extending the write path all the way to the end user. When a client is first initialized, it would still need to use a read path to get its initial state, but thereafter it could rely on a stream of state changes sent by the server.

#### Reads are events too





