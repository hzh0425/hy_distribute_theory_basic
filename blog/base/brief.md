分布式系统中除了CAP理论，还有一个不得不说的BASE理论，这不仅是面试中常问的一个知识点，也是在学习分布式系统时候一个绕不过去的基础。

1、CAP理论回顾

分布式CAP理论告诉我们一个分布式系统最多只能同时满足一致性（Consistency）、可用性（Availability）和分区容忍 性（Partition tolerance）这三项中的两项。在这三项当中AP在实际应用中较多，它舍弃了一致性。

为什么要舍弃一致性呢？就好比是我们在买火车票的时候，明明看到还有一张票，可是等我选好了座位准备付钱的时候，系统却提示没票了。这就是舍弃了一致性，数据可能是不一致的。但是分区容错性和可用性却得到了满足。

但这不是说一致性不重要，相反恰恰它是最重要的。对我们来说，我们舍弃的只是强一致性。但是一定要满足最终一致性。也就是说，但是最终也要将数据同步成功来保证数据一致。而强一致性，要求在任何时间查询每个节点数据都必须一致。

2、Base理论介绍

BASE 是 Basically Available(基本可用)、Soft state(软状态)和 Eventually consistent (最终一致性)三个短语的缩 写。BASE理论是对CAP中AP的一个扩展。下面我们来介绍一下这三个概念。

（1）基本可用

指分布式系统在出现故障的时候，保证核心可用，允许损失部分可用性。例如，电商在做促销时，为了保证购物系统的稳定性，部分消费者可能会被引导到一个降级的页面。

（2）软状态

指允许系统中的数据存在中间状态，并认为该中间状态不会影响系统整体可用性，即允许系统不同节点的数据副本之间进行同步的过程存在时延。就好比是使用支付宝的时候，会出现支付中、数据同步中等状态，这时候就叫做软状态。但是最终会显示支付成功。

（3）最终一致性

最终一致性强调的是系统中的数据副本，在经过一段时间的同步后，最终能达到一致的状态。如订单的"支付中"状态，最终会变 为“支付成功”或者"支付失败"，使订单状态与实际交易结果达成一致，但需要一定时间的延迟、等待。

![img](https://gitee.com/zisuu/picture/raw/master/img/20210219160033.jpeg)