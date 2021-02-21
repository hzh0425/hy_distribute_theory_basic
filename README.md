# 1.MIT分布式系统课程

网课地址:https://www.bilibili.com/video/BV1qk4y197bB?from=search&seid=7469911290247608861

官网资源:http://nil.csail.mit.edu/6.824/2020/schedule.html

## 课程内容

### 第一课:[lab1-introduction](doc/mit/lab1-introduction.md)

**前置理论及论文基础**

- **cap**

1.[cap简介](blog/cap/brief.md)

2.[cap-theorem-revisited](paper/cap/cap-theorem-revisited.md)

- **base**

1.[base简介](blog/base/brief.md)

2.[Base: An Acid Alternative](paper/base/base-danPritichett.md)

3.[分布式事务简介](blog/transaction/type.md)

- **mapReduce**

[google:mapreduce](paper/google/mapreduce.md)

### 第二课:[lab2-gfs文件系统](doc/mit/lab2-gfs.md)

**前置理论及论文基础**

- gfs论文

[google:gfs](paper/google/gfs.md)

### 第三课:[lab3-VMware FT](doc/mit/lab3-VMware FT.md)

**前置理论及论文基础**

- [VMware FT](https://pdos.csail.mit.edu/6.824/papers/vm-ft.pdf)

# 2.论文借鉴

**概览文章**

[http://www.rgoarchitects.com/Files/fallacies.pdf](https://link.zhihu.com/?target=http%3A//www.rgoarchitects.com/Files/fallacies.pdf) CS7680著名的9个论述 也是这门课推荐对于分布式系统的一个初步认识

[http://citeseerx.ist.psu.edu/viewdoc/download;jsessionid=20A79A6520D69264C29248D0387C6703?doi=10.1.1.209.355&rep=rep1&type=pdf](https://link.zhihu.com/?target=http%3A//citeseerx.ist.psu.edu/viewdoc/download%3Bjsessionid%3D20A79A6520D69264C29248D0387C6703%3Fdoi%3D10.1.1.209.355%26rep%3Drep1%26type%3Dpdf) windows live的架构师james总结一系列大型后台服务的设计原则

## **CAP**

[https://robertgreiner.com/cap-theorem-revisited/](https://link.zhihu.com/?target=https%3A//robertgreiner.com/cap-theorem-revisited/) 准确说是一篇blog，很精简，文字也不多，其实文中的图比文字更清晰。cap的理解也经历了一些纠结的过程，这一篇其实是作者多年后的二次理解。所以出错其实没啥问题，这位老板就完全推翻了之前文章里的阐述

[http://ksat.me/a-plain-english-introduction-to-cap-theorem](https://link.zhihu.com/?target=http%3A//ksat.me/a-plain-english-introduction-to-cap-theorem) 也是通俗易懂的入门介绍cap的blog

[https://www.infoq.com/articles/cap-twelve-years-later-how-the-rules-have-changed/](https://link.zhihu.com/?target=https%3A//www.infoq.com/articles/cap-twelve-years-later-how-the-rules-have-changed/) brewer多年以后写的关于cap的一些误解，C和A并不是完全对立的状态

[https://sookocheff.com/post/databases/cap-twelve-years-later/](https://link.zhihu.com/?target=https%3A//sookocheff.com/post/databases/cap-twelve-years-later/) 是对上面这片文章的review心得



[http://citeseerx.ist.psu.edu/viewdoc/download?doi=10.1.1.24.3690&rep=rep1&type=pdf](https://link.zhihu.com/?target=http%3A//citeseerx.ist.psu.edu/viewdoc/download%3Fdoi%3D10.1.1.24.3690%26rep%3Drep1%26type%3Dpdf) 开始用了两个新名词来阐述

A)yield, which is the probability of completing a request .感觉说的就是A

B)harvest ,measures the fraction of the data reflected in the response.感觉说的就是C

这篇论文对于available提出里两个比较好的方案：

1)牺牲harvest换来yield

2）应用架构拆分 和 正交机制

**BASE**

[https://queue.acm.org/detail.cfm?id=1394128](https://link.zhihu.com/?target=https%3A//queue.acm.org/detail.cfm%3Fid%3D1394128) base一致性的开山鼻祖，首次提出了和acid相反的一种理论，论文中给出了一些单机事务到多机事务的演进过程，并没有觉得很理论，工程很值得借鉴

## 一致性

[http://jepsen.io/consistency#fundamental-concepts](https://link.zhihu.com/?target=http%3A//jepsen.io/consistency%23fundamental-concepts) 一致性的模型，高屋建瓴，是一篇blog

[http://vukolic.com/consistency-survey.pdf](https://link.zhihu.com/?target=http%3A//vukolic.com/consistency-survey.pdf) 概述的文章 先看看

- **sequential consistency**

[https://lamport.azurewebsites.net/pubs/lamport-how-to-make.pdf](https://link.zhihu.com/?target=https%3A//lamport.azurewebsites.net/pubs/lamport-how-to-make.pdf) lamport大神不用过多的介绍，读他的论文唯一的感受就是智商的差别吧

[https://cs.brown.edu/~mph/HerlihyW90/p463-herlihy.pdf](https://link.zhihu.com/?target=https%3A//cs.brown.edu/~mph/HerlihyW90/p463-herlihy.pdf) 也是线性一致性的文章 作者在cmu发表的

- **eventual consistency**

最终一致性的文章首推 aws的cto

[https://www.allthingsdistributed.com/2008/12/eventually_consistent.html](https://link.zhihu.com/?target=https%3A//www.allthingsdistributed.com/2008/12/eventually_consistent.html)

[https://www.cs.tau.ac.il/~mad/publications/podc2015-replds.pdf](https://link.zhihu.com/?target=https%3A//www.cs.tau.ac.il/~mad/publications/podc2015-replds.pdf) 讲了一些高可用和一致性之间的trade-off

[https://pdfs.semanticscholar.org/6877/32ca90ce8ec57c0ec8530863b8a693bf4f51.pdf](https://link.zhihu.com/?target=https%3A//pdfs.semanticscholar.org/6877/32ca90ce8ec57c0ec8530863b8a693bf4f51.pdf) 描述了 最终一致性 和 因果一致性的关系

[https://haslab.uminho.pt/tome/files/global_logical_clocks.pdf](https://link.zhihu.com/?target=https%3A//haslab.uminho.pt/tome/files/global_logical_clocks.pdf)

- **causal consistency**

[http://www.bailis.org/papers/bolton-sigmod2013.pdf](https://link.zhihu.com/?target=http%3A//www.bailis.org/papers/bolton-sigmod2013.pdf) Bolt-on的架构设计

[https://www.cs.cmu.edu/~dga/papers/cops-sosp2011.pdf](https://link.zhihu.com/?target=https%3A//www.cs.cmu.edu/~dga/papers/cops-sosp2011.pdf) cops的架构设计

[https://www.cs.princeton.edu/~wlloyd/papers/eiger-nsdi13.pdf](https://link.zhihu.com/?target=https%3A//www.cs.princeton.edu/~wlloyd/papers/eiger-nsdi13.pdf)

[https://www.ronpub.com/OJDB_2015v2i1n02_Elbushra.pdf](https://link.zhihu.com/?target=https%3A//www.ronpub.com/OJDB_2015v2i1n02_Elbushra.pdf) 一个causal consistency的db设计与实现

[https://smartech.gatech.edu/bitstream/handle/1853/6781/GIT-CC-93-55.pdf?sequence=1&isAllowed=y](https://link.zhihu.com/?target=https%3A//smartech.gatech.edu/bitstream/handle/1853/6781/GIT-CC-93-55.pdf%3Fsequence%3D1%26isAllowed%3Dy)

从前三篇文章的作者来看，ucb & cmu&priceton 还是很值得一读的

最后一篇的年代已经久远，其实发现计算机的一些理论基础其实是很经得起时间的考验的，所以码农其实也可以过的没有那么的有危机感^_^

[https://pdfs.semanticscholar.org/7725/8064a686dfd61d7232172d6706711606dcfc.pdf](https://link.zhihu.com/?target=https%3A//pdfs.semanticscholar.org/7725/8064a686dfd61d7232172d6706711606dcfc.pdf) 这个是最后一篇论文的ppt版本

[https://arxiv.org/pdf/1802.00706.pdf](https://link.zhihu.com/?target=https%3A//arxiv.org/pdf/1802.00706.pdf)

- **weak consistency**

[http://pmg.csail.mit.edu/papers/adya-phd.pdf](https://link.zhihu.com/?target=http%3A//pmg.csail.mit.edu/papers/adya-phd.pdf)

## 分布式锁

[https://static.googleusercontent.com/media/research.google.com/zh-CN//archive/chubby-osdi06.pdf](https://link.zhihu.com/?target=https%3A//static.googleusercontent.com/media/research.google.com/zh-CN//archive/chubby-osdi06.pdf) Google出品的chubby 必属精品

[https://www.usenix.org/legacy/event/atc10/tech/full_papers/Hunt.pdf](https://link.zhihu.com/?target=https%3A//www.usenix.org/legacy/event/atc10/tech/full_papers/Hunt.pdf) Yahoo的zookeeper

## 分布式kv存储

[http://static.googleusercontent.com/media/research.google.com/en//archive/bigtable-osdi06.pdf](https://link.zhihu.com/?target=http%3A//static.googleusercontent.com/media/research.google.com/en//archive/bigtable-osdi06.pdf) Google三驾马车之一bigtable,hbase的蓝本

[http://static.googleusercontent.com/media/research.google.com/en/us/archive/gfs-sosp2003.pdf](https://link.zhihu.com/?target=http%3A//static.googleusercontent.com/media/research.google.com/en/us/archive/gfs-sosp2003.pdf) Google三架马车之二gfs，hdfs的蓝本

[https://static.googleusercontent.com/media/research.google.com/en//archive/bigtable-osdi06.pdf](https://link.zhihu.com/?target=https%3A//static.googleusercontent.com/media/research.google.com/en//archive/bigtable-osdi06.pdf) Google三架马车之三bigtable，hbase的蓝本

[https://www.allthingsdistributed.com/files/amazon-dynamo-sosp2007.pdf](https://link.zhihu.com/?target=https%3A//www.allthingsdistributed.com/files/amazon-dynamo-sosp2007.pdf) 现代很多的kv设计或多或少的都参考了先驱dynamo的设计，值得刷10遍以上。[读后感](https://zhuanlan.zhihu.com/p/140050721)

[https://www.cs.cornell.edu/Projects/ladis2009/papers/Lakshman-ladis2009.PDF](https://link.zhihu.com/?target=https%3A//www.cs.cornell.edu/Projects/ladis2009/papers/Lakshman-ladis2009.PDF) 2009年Cassandra设计的论文 ，很多思想借鉴了dynamo，对于一致性哈希的吐槽也高度类似。

在replication的过程中，也会通过一个coordinator节点（master节点）来对其他节点进行replicate（这一点和dynamo一样），但是Cassandra提供了一系列的replicate policy可以选择，比如 Rack Unaware, Rack Aware (within a datacenter) and Datacenter Aware. Cassandra也沿用了dynamo里面关于preference list的定义



[https://www.ssrc.ucsc.edu/media/pubs/9c7bcd06ff4eeccef2cb4c7813fe33ba7d4805c7.pdf](https://link.zhihu.com/?target=https%3A//www.ssrc.ucsc.edu/media/pubs/9c7bcd06ff4eeccef2cb4c7813fe33ba7d4805c7.pdf)

[https://dsf.berkeley.edu/jmh/papers/anna_ieee18.pdf](https://link.zhihu.com/?target=https%3A//dsf.berkeley.edu/jmh/papers/anna_ieee18.pdf) ucb出的一篇高性能的kv存储，号称比redis快几十倍，使用coordination-free consistency models。虽然说是特别快，但是其实业界的是用并不广泛

[https://cs.ulb.ac.be/public/_media/teaching/influxdb_2017.pdf](https://link.zhihu.com/?target=https%3A//cs.ulb.ac.be/public/_media/teaching/influxdb_2017.pdf) 时间序列的数据库的一篇介绍 ，介绍了几个应用场景 iot ebay等 ，influxdb的介绍

[http://btw2017.informatik.uni-stuttgart.de/slidesandpapers/E4-14-109/paper_web.pdf](https://link.zhihu.com/?target=http%3A//btw2017.informatik.uni-stuttgart.de/slidesandpapers/E4-14-109/paper_web.pdf) 比较了业界的几种TSDB的异同





无论是kv还是传统的关系型数据库，在分布式系统里面无非都会涉及到以下这几方面

- replication

[https://dsf.berkeley.edu/cs286/papers/dangers-sigmod1996.pdf](https://link.zhihu.com/?target=https%3A//dsf.berkeley.edu/cs286/papers/dangers-sigmod1996.pdf) 指出了一种在replication中存在的问题，并给出了解决方案

- partition&shard

分区都逃不了一致性哈希，

[https://www.akamai.com/us/en/multimedia/documents/technical-publication/consistent-hashing-and-random-trees-distributed-caching-protocols-for-relieving-hot-spots-on-the-world-wide-web-technical-publication.pdf](https://link.zhihu.com/?target=https%3A//www.akamai.com/us/en/multimedia/documents/technical-publication/consistent-hashing-and-random-trees-distributed-caching-protocols-for-relieving-hot-spots-on-the-world-wide-web-technical-publication.pdf) 被引用度特别高的一篇文章,但是这个版本也是被吐槽最多的，dynamo吐槽过，Cassandra也吐槽了一把

1）First, the random position assignment of each node on the ring leads to non-uniform data and load distribution.

2）Second, the basic algorithm is oblivious to the heterogeneity in the performance of nodes.

解决方案

1）One is for nodes to get assigned to multiple positions in the circle (like in Dynamo) dynamo用的就是这种方法

2）the second is to analyze load information on the ring and have lightly loaded nodes move on the ring to alleviate heavily loaded nodes 这种方法被Cassandra采用

[http://citeseerx.ist.psu.edu/viewdoc/download?doi=10.1.1.217.2218&rep=rep1&type=pdf](https://link.zhihu.com/?target=http%3A//citeseerx.ist.psu.edu/viewdoc/download%3Fdoi%3D10.1.1.217.2218%26rep%3Drep1%26type%3Dpdf) 2）用的方法 也就是这片论文提出的方法

- memship
- failure detect
- updated conflicts
- implement

关于实现[http://www.sosp.org/2001/papers/welsh.pdf](https://link.zhihu.com/?target=http%3A//www.sosp.org/2001/papers/welsh.pdf) 这篇论文的出镜率特别高，里面的思想被Cassandra和dynamo都采用了 ，作者也是提出cap的大神Eric Brewer（第三作者），值得反复研读



[https://storage.googleapis.com/pub-tools-public-publication-data/pdf/03de87e2856b06a94ffae7dca218db2d4b9afd39.pdf](https://link.zhihu.com/?target=https%3A//storage.googleapis.com/pub-tools-public-publication-data/pdf/03de87e2856b06a94ffae7dca218db2d4b9afd39.pdf) 这个是2019年Google提出的一种有状态的kv存储的思路。在工业界的下个请求依赖于上一个请求的情况

**数据库**

- **查询优化器**

[https://arxiv.org/pdf/1608.02611.pdf](https://link.zhihu.com/?target=https%3A//arxiv.org/pdf/1608.02611.pdf)

[https://dl.acm.org/doi/pdf/10.1145/335191.335451](https://link.zhihu.com/?target=https%3A//dl.acm.org/doi/pdf/10.1145/335191.335451)

[https://dl.acm.org/doi/pdf/10.1145/2304510.2304525](https://link.zhihu.com/?target=https%3A//dl.acm.org/doi/pdf/10.1145/2304510.2304525)





**MQ**

- **kafka**

[http://notes.stephenholiday.com/Kafka.pdf](https://link.zhihu.com/?target=http%3A//notes.stephenholiday.com/Kafka.pdf) 现在很火的kafa最初设计的论文，细节有些已经被优化，基本的架构还是很值得反复研读。比如

In general, Kafka only guarantees at-least-once delivery. Exactly once delivery typically requires two-phase commits and is not necessary for our applications

最初kafka只是支持at-least的delivery, 但是不支持exactly once的投递，具体哪个版本开始支持有点记不清了



**分布式文件系统**

除了大名鼎鼎的gfs 分布式文件系统已经走过了好几十个年头了

[https://www.cs.cmu.edu/~satya/docdir/satya-ieeetc-coda-1990.pdf](https://link.zhihu.com/?target=https%3A//www.cs.cmu.edu/~satya/docdir/satya-ieeetc-coda-1990.pdf) 1990年的coda，在很多的论文中出镜率非常高，后面的fs也借鉴了coda的一些思想



**分布式事务&事务隔离级别**

[http://www.vldb.org/pvldb/vol7/p181-bailis.pdf](https://link.zhihu.com/?target=http%3A//www.vldb.org/pvldb/vol7/p181-bailis.pdf) 引用率很高的一篇文章 这里面也引用了下面的这篇文章中关于事务隔离级别P0，P1的引用，看之前可以先看下面这篇文章。比如，脏写，脏读，不可重复读&fuzzy读，幻读等

读未提交保证了写的串行化，注意只是写的串行化（并不能保证读写的串行化，依然有可能产生脏读），下面这篇论文里面是避免了脏写的操作。如何处理写的冲突呢？ 打时间戳或者last write win的方式都是可行的

[https://www.microsoft.com/en-us/research/wp-content/uploads/2016/02/tr-95-51.pdf](https://link.zhihu.com/?target=https%3A//www.microsoft.com/en-us/research/wp-content/uploads/2016/02/tr-95-51.pdf) 不管是怎么讲事务隔离级别，最原生的味道是这一篇，其他的文章都是咀嚼过吐出来的

其中也参考了 [https://pdfs.semanticscholar.org/b40a/2bed6469ccea11d1c5f884215805ba785019.pdf?_ga=2.256890123.1236479272.1592556133-1143139955.1585301775](https://link.zhihu.com/?target=https%3A//pdfs.semanticscholar.org/b40a/2bed6469ccea11d1c5f884215805ba785019.pdf%3F_ga%3D2.256890123.1236479272.1592556133-1143139955.1585301775) 里面阐述了很多隔离级别的标准

**共识算法**



[https://lamport.azurewebsites.net/pubs/paxos-simple.pdf](https://link.zhihu.com/?target=https%3A//lamport.azurewebsites.net/pubs/paxos-simple.pdf) paxos的simple版本，原来的版本太晦涩，lamport大神自己可能发现之前写的太高深了，写了一个通俗易懂的版本

[https://dl.acm.org/doi/pdf/10.1145/3373376.3378496](https://link.zhihu.com/?target=https%3A//dl.acm.org/doi/pdf/10.1145/3373376.3378496) hermes

[https://raft.github.io/raft.pdf](https://link.zhihu.com/?target=https%3A//raft.github.io/raft.pdf) 这个是精简版的raft 里面有些概念如果理解起来吃力可以看下作者的博士毕业论文[https://github.com/ongardie/dissertation#readme](https://link.zhihu.com/?target=https%3A//github.com/ongardie/dissertation%23readme) 里面有download的连接，以下的几篇文章都是raft的推荐

[https://www.cl.cam.ac.uk/~ms705/pub/papers/2015-osr-raft.pdf](https://link.zhihu.com/?target=https%3A//www.cl.cam.ac.uk/~ms705/pub/papers/2015-osr-raft.pdf) raft 的分析文章

[http://verdi.uwplse.org/raft-proof.pdf](https://link.zhihu.com/?target=http%3A//verdi.uwplse.org/raft-proof.pdf)

[http://verdi.uwplse.org/verdi.pdf](https://link.zhihu.com/?target=http%3A//verdi.uwplse.org/verdi.pdf) verdi的实现

[https://www.cl.cam.ac.uk/techreports/UCAM-CL-TR-857.pdf](https://link.zhihu.com/?target=https%3A//www.cl.cam.ac.uk/techreports/UCAM-CL-TR-857.pdf) raft一致性的分析

[https://hal.inria.fr/hal-01086522/document](https://link.zhihu.com/?target=https%3A//hal.inria.fr/hal-01086522/document)



名字服务

[http://static.cs.brown.edu/courses/csci2270/archives/2012/papers/replication/hunt.pdf](https://link.zhihu.com/?target=http%3A//static.cs.brown.edu/courses/csci2270/archives/2012/papers/replication/hunt.pdf) zk最初设计的论文，感觉比市面上的一些中文材料好懂，推荐