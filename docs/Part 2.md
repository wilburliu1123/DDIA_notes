For a successful technology, reality must take precedence over public relations, for nature cannot be fooled.

—Richard Feynman

这句话到底是什么意思？ 

这本书的第一部分讨论的都是一台机器的情况，现在第二部分要更进一步提出一个问题：
> 如果多台机器都要进行数据存储+retrieval, 会发生什么？

我们有很多理由把一个数据库存在多台机器上面：
*Scalability* 
当一台机器支持不了目前的数据量的时候，据需要多台机器来分担数据量
*Fault tolerance/high availability*
这个应该是绝大多数的目的，比如你一台机器坏了，其他机器可以替他接受请求
*Latency*
通过一个物理上更近的距离来降低 latency (北京到纽约光都需要走 $10,988.88 km \div 299 792 .458 km / s$ = 36.65496 ms)

### Scaling to Higher Load
如果只是需要 scale to higher load. 那么我们买更好的机器就可以了，也叫做 *scaling up*/*vertical scaling*

更多的CPU，更大的 RAM，更多个硬盘。这些硬件资源通常都在一个 OS 下面管理，所以也还算做 single machine. 也叫 *shared memory architecture* 
但这种方式的缺点在于 cost 可以上长得很快（几何级？反正不是线性增加的）

我记得 oracle 就有自己的 hardware，而且貌似及其贵……

还有一种方式是 *share-disk architecture*, 就是多个机器通过网络共享 array of disk。 这个scaling的方式也有limit (contention and locking)

#### Shared nothing architectures
*Shared nothing architectures* (也叫 *horizontal scaling* or *scaling out*) 是更流行的方式
在这个 scaling method中，每一台机器或者是 virtual machine is called *node*。每一个node 都是独立的，都需要通过 network 来完成协作

shared nothing system 并不需要特殊的硬件，所以你可以买任何适用的机器 (几台 resberry pi 应该也是可以的) 你还可以把数据分布到不同的地域，reduce latency for users 

有了现在的云服务，小公司一样可以建一个 multi-region 的分布式系统

这本书主要集中讲 share nothing architectures, 并不是因为他是最好的选择而是因为他需要最多的 care，需要大量的engineering effort

尽管 shared nothing 有很多好处，但也带来了极大的复杂度，下面几章主要讨论 shared nothing architecture 的问题
### Replication vs Partitioning
主要有2种把数据分到多个机器上面
*Replication*
把同样的数据复制到多个机器上 chapter 5 讲这一个方法
*Partitioning*
把一个大的数据库切分成多个 subsets (也叫 *partitions*/*sharding*) chapter 6 talks about partitions
不过这两个一般是一起用的
![[partition_plus_replication.png]]

chapter 7 讨论 transactions (也是 shared-nothing system 会遇到的各种问题，以及解决方案)
最后 chapter 8 and chapter 9 讲 distributed systems 的根本限制


