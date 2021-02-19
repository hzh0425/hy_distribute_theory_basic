**Note:** *Close to two months ago, I wrote a [blog post explaining the CAP Theorem](http://robertgreiner.com/2014/06/cap-theorem-explained/). Since publishing, I've come to realize that my thinking on the subject was quite outdated and is no longer applicable to the real world. I've attempted to make up for that in this post.*

In today's technical landscape, we are witnessing a strong and increasing desire to scale systems *out* when additional resources (compute, storage, etc.) are needed to successfully complete workloads in a reasonable time frame. This is accomplished through adding additional commodity hardware to a system to handle the increased load. As a result of this scaling strategy, an additional penalty of complexity is incurred in the system. This is where the CAP theorem comes into play.

The CAP Theorem states that, in a distributed system (a collection of interconnected nodes that share data.), you can only have two out of the following three guarantees across a write/read pair: Consistency, Availability, and Partition Tolerance - one of them must be sacrificed. However, as you will see below, you don't have as many options here as you might think.

![img](https://robertgreiner.com/content/images/2019/09/CAP-overview.png)

- **Consistency** - A read is guaranteed to return the most recent write for a given client.
- **Availability** - A non-failing node will return a reasonable response within a reasonable amount of time (no error or timeout).
- **Partition Tolerance** - The system will continue to function when network partitions occur.

Before moving further, we need to set one thing straight. Object Oriented Programming != Network Programming! There are assumptions that we take for granted when building applications that share memory, which break down as soon as nodes are split across space and time.

One such [*fallacy of distributed computing*](http://en.wikipedia.org/wiki/Fallacies_of_Distributed_Computing) is that networks are reliable. They aren't. Networks and parts of networks go down frequently and unexpectedly. Network failures *happen to your system* and you don't get to choose when they occur.

Given that networks aren't completely reliable, you must tolerate partitions in a distributed system, period. Fortunately, though, you get to choose what to do when a partition does occur. According to the CAP theorem, this means we are left with two options: Consistency and Availability.

- **CP** - Consistency/Partition Tolerance - Wait for a response from the partitioned node which could result in a timeout error. The system can also choose to return an error, depending on the scenario you desire. Choose Consistency over Availability when your business requirements dictate atomic reads and writes.

![img](https://robertgreiner.com/content/images/2019/09/CAP-CP.png)

- **AP** - Availability/Partition Tolerance - Return the most recent version of the data you have, which could be stale. This system state will also accept writes that can be processed later when the partition is resolved. Choose Availability over Consistency when your business requirements allow for some flexibility around when the data in the system synchronizes. Availability is also a compelling option when the system needs to continue to function in spite of external errors (shopping carts, etc.)

![img](https://robertgreiner.com/content/images/2019/09/CAP-AP.png)

The decision between Consistency and Availability is a *software trade off*. You can choose what to do in the face of a network partition - the control is in your hands. Network outages, both temporary and permanent, are a fact of life and occur whether you want them to or not - this exists outside of your software.

Building distributed systems provide many advantages, but also adds complexity. Understanding the trade-offs available to you in the face of network errors, and choosing the right path is vital to the success of your application. Failing to get this right from the beginning could doom your application to failure before your first deployment.









读后感:

> - 文章开头讲述cap理论可以解决什么问题:不断空充机器时,造成的复杂度后果
> - 接着讲述CAP理论的具体内容
> - 随后,阐述网络分区一定会存在的原因:网络不可靠,经常崩溃
> - 也即P一定要存在,但幸运的是,可以在网络分区出现时,我们能做什么(也即在A和C中选一个)
> - 最后就讲述了在什么情况下选择A或者C:
>   - 当业务要求原子性读写时->C,可以直接返回错误,但会使系统不可用
>   - 当业务在数据同步时能允许弹性选择->A,但会造成数据不一致