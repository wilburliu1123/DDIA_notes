Part 1 and part 2 discuss about all background knowledge about distributed database system. Based on the assumption about system only use one particular type of database

In practice, data systems often made up from different kinds of databases. In this last part of the book, we will discuss about integrate different data systems into one application architecture. In reality, integrating disparate systems is one of the most important things that needs to be done in nontrivial application 

Data system can derive into 2 categories
*System of records* 
	System of record can be seen as *source of truth*. It holds authoritative version of your data. 

*Derived data systems*
	Data in derived system is result of transforming or processing your data in some way

Derived data can be seen as denormalized or redundant data. Enabling you to look at the data from different "point of view"

Not all system make clear distinction between systems of record and derived data but it is good distinction to make. Because it clarifies data flow of your system. Being explicit about which part of the system have which input and which output and how they depend on each other 

Data systems (storage engines, query languages) are not inherently system of record or derived systems. Just like a flat head screw driver also works on cross nails. Database is just a tool, *you* define how it is used in your application. Either be a system of record or derived data

Being clear about which data derived from other data is important for system architecture. This point will be continue throughout this part of the book

[[Chapter 10]] will review batch-oriented dataflow systems such as MapReduce and good tools and principles for building large scale data systems
[[Chapter 11]] will took those ideas and apply them in streaming, which allow us to do same kinds of things with lower delays 
[[Chapter 12]] concludes the book by exploring those ideas to build reliable, scalable, and maintainable application in the future

