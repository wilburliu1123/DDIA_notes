DDIA
这本书把数据系统分为了3部分来进行讨论

1. Part 1 讨论了设计大数据系统的基本概念。[[Chapter 1]] 讨论了我们最终的目标是什么 *reliability* (可靠性), *scalability* (可扩展性), 还有 *maintainability* (可维护性). 我们应该如何思考以及实现这些概念 Chapter 2 讨论了 data model and query languages (数据模型以及查询语言) 看他们在什么场景下下最适用。 Chapter 3 讨论了 storage engine (存储引擎)。为了更有效的查找出我们想要的数据，数据库是如何在硬盘上存储数据的。 Chapter 4 主要讲述数据是如何编码的(serialization)，以及 evolution of schemas over time
2. Part 2 从之前一台机器转换到了多台机器，也就是 distributed system. 这是scalability的前提，这一部分讲了 distributed system 的很多概念, replication (Chapter 5), partitioning/sharding (Chapter 6), and transactions (Chapter 7). 之后讲了 distributed system 的诸多问题 (Chapter 8) 以及在分布式系统里面实现 consistency and consensus 究竟意味着什么 (Chapter 9)
3. 第三部分讨论了衍生数据 (datasets derived from other datasets)。衍生数据 (derived datasets) 通常意味着整合不同的DB，caches, indexes. Chapter 10 从 batch process 的角度讨论衍生数据，在这个基础上开始讨论 stream processing (Chapter 11). 最后第12章把所有概念整合起来讨论以后的 distributed system

This notes is for discussion only
