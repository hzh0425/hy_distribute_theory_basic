## 一 Chain Replication

这一部分，我们来讨论另一个论文CRAQ（Chain Replication with Apportioned Queries）。我们选择CRAQ论文有两个原因：第一个是它通过复制实现了容错；第二是它通过以链复制API请求这种有趣的方式，提供了与Raft相比不一样的属性。

CRAQ是对于一个叫链式复制（Chain Replication）的旧方案的改进。Chain Replication实际上用的还挺多的，有许多现实世界的系统使用了它，CRAQ是对它的改进。CRAQ采用的方式与Zookeeper非常相似，它通过将读请求分发到任意副本去执行，来提升读请求的吞吐量，所以副本的数量与读请求性能成正比。CRAQ有意思的地方在于，它在任意副本上执行读请求的前提下，还可以保证线性一致性（Linearizability）。这与Zookeeper不太一样，Zookeeper为了能够从任意副本执行读请求，不得不牺牲数据的实时性，因此也就不是线性一致的。CRAQ却可以从任意副本执行读请求，同时也保留线性一致性，这一点非常有趣。



![img](https://pic3.zhimg.com/v2-f8ad56fc79401d846c46114c254c76da_b.jpg)



首先，我想讨论旧的Chain Replication系统。Chain Replication是这样一种方案，你有多个副本，你想确保它们都看到相同顺序的写请求（这样副本的状态才能保持一致），这与Raft的思想是一致的，但是它却采用了与Raft不同的拓扑结构。

首先，在Chain Replication中，有一些服务器按照链排列。第一个服务器称为HEAD，最后一个被称为TAIL。



![img](https://pic1.zhimg.com/v2-5968a01c3bc7133c92ac53177d3737a8_b.jpg)



当客户端想要发送一个写请求，写请求总是发送给HEAD。



![img](https://pic1.zhimg.com/v2-c5f8a22986f3dcce68ac5841ff22ed84_b.jpg)



HEAD根据写请求更新本地数据，我们假设现在是一个支持PUT/GET的key-value数据库。所有的服务器本地数据都从A开始。



![img](https://pic4.zhimg.com/v2-a8cfb16c989b6ba1a487d0d1679b2423_b.jpg)



当HEAD收到了写请求，将本地数据更新成了B，之后会再将写请求通过链向下一个服务器传递。



![img](https://pic1.zhimg.com/v2-00643f986dc4033d9602ab4dc0183db0_b.jpg)



下一个服务器执行完写请求之后，再将写请求向下一个服务器传递，以此类推，所有的服务器都可以看到写请求。



![img](https://pic2.zhimg.com/v2-864ac9e09bdda416ca30a44b8dd8a705_b.jpg)



当写请求到达TAIL时，TAIL将回复发送给客户端，表明写请求已经完成了。这是处理写请求的过程。



![img](https://pic1.zhimg.com/v2-6804ad85e9c46289312d5fee2f421024_b.jpg)



对于读请求，如果一个客户端想要读数据，它将读请求发往TAIL，



![img](https://pic4.zhimg.com/v2-d6d5ca7cf95739c36eab40291b00328b_b.jpg)



TAIL直接根据自己的当前状态来回复读请求。所以，如果当前状态是B，那么TAIL直接返回B。读请求处理的非常的简单。



![img](https://pic2.zhimg.com/v2-0586b322c99eb668d227e4ba6abd6fe9_b.jpg)



这里只是Chain Replication，并不是CRAQ。Chain Replication本身是线性一致的，在没有故障时，从一致性的角度来说，整个系统就像只有TAIL一台服务器一样，TAIL可以看到所有的写请求，也可以看到所有的读请求，它一次只处理一个请求，读请求可以看到最新写入的数据。如果没有出现故障的话，一致性是这么得到保证的，非常的简单。

从一个全局角度来看，除非写请求到达了TAIL，否则一个写请求是不会commit，也不会向客户端回复确认，也不能将数据通过读请求暴露出来。而为了让写请求到达TAIL，它需要经过并被链上的每一个服务器处理。所以我们知道，一旦我们commit一个写请求，一旦向客户端回复确认，一旦将写请求的数据通过读请求暴露出来，那意味着链上的每一个服务器都知道了这个写请求。

## 二 Fail Recover

在Chain Replication中，出现故障后，你可以看到的状态是相对有限的。因为写请求的传播模式非常有规律，我们不会陷入到类似于Raft论文中图7和图8描述的那种令人毛骨悚然的复杂场景中。并且在出现故障之后，也不会出现不同的副本之间各种各样不同步的场景。

在Chain Replication中，因为写请求总是依次在链中处理，写请求要么可以达到TAIL并commit，要么只到达了链中的某一个服务器，之后这个服务器出现故障，在链中排在这个服务器后面的所有其他服务器不再能看到写请求。所以，只可能有两种情况：committed的写请求会被所有服务器看到；而如果一个写请求没有commit，那就意味着在导致系统出现故障之前，写请求已经执行到链中的某个服务器，所有在链里面这个服务器之前的服务器都看到了写请求，所有在这个服务器之后的服务器都没看到写请求。

总的来看，Chain Replication的故障恢复也相对的更简单。

如果HEAD出现故障，作为最接近的服务器，下一个节点可以接手成为新的HEAD，并不需要做任何其他的操作。对于还在处理中的请求，可以分为两种情况：

- 对于任何已经发送到了第二个节点的写请求，不会因为HEAD故障而停止转发，它会持续转发直到commit。
- 如果写请求发送到HEAD，在HEAD转发这个写请求之前HEAD就故障了，那么这个写请求必然没有commit，也必然没有人知道这个写请求，我们也必然没有向发送这个写请求的客户端确认这个请求，因为写请求必然没能送到TAIL。所以，对于只送到了HEAD，并且在HEAD将其转发前HEAD就故障了的写请求，我们不必做任何事情。或许客户端会重发这个写请求，但是这并不是我们需要担心的问题。

如果TAIL出现故障，处理流程也非常相似，TAIL的前一个节点可以接手成为新的TAIL。所有TAIL知道的信息，TAIL的前一个节点必然都知道，因为TAIL的所有信息都是其前一个节点告知的。

中间节点出现故障会稍微复杂一点，但是基本上来说，需要做的就是将故障节点从链中移除。或许有一些写请求被故障节点接收了，但是还没有被故障节点之后的节点接收，所以，当我们将其从链中移除时，故障节点的前一个节点或许需要重发最近的一些写请求给它的新后继节点。这是恢复中间节点流程的简单版本。

Chain Replication与Raft进行对比，有以下差别：

- 从性能上看，对于Raft，如果我们有一个Leader和一些Follower。Leader需要直接将数据发送给所有的Follower。所以，当客户端发送了一个写请求给Leader，Leader需要自己将这个请求发送给所有的Follower。然而在Chain Replication中，HEAD只需要将写请求发送到一个其他节点。数据在网络中发送的代价较高，所以Raft Leader的负担会比Chain Replication中HEAD的负担更高。当客户端请求变多时，Raft Leader会到达一个瓶颈，而不能在单位时间内处理更多的请求。而同等条件以下，Chain Replication的HEAD可以在单位时间处理更多的请求，瓶颈会来的更晚一些。
- 另一个与Raft相比的有趣的差别是，Raft中读请求同样也需要在Raft Leader中处理，所以Raft Leader可以看到所有的请求。而在Chain Replication中，每一个节点都可以看到写请求，但是只有TAIL可以看到读请求。所以负载在一定程度上，在HEAD和TAIL之间分担了，而不是集中在单个Leader节点。
- 前面分析的故障恢复，Chain Replication也比Raft更加简单。这也是使用Chain Replication的一个主要动力。

> 学生提问：如果一个写请求还在传递的过程中，还没有到达TAIL，TAIL就故障了，会发生什么？
> Robert教授：如果这个时候TAIL故障了，TAIL的前一个节点最终会看到这个写请求，但是TAIL并没有看到。因为TAIL的故障，TAIL的前一个节点会成为新的TAIL，这个写请求实际上会完成commit，因为写请求到达了新的TAIL。所以新的TAIL可以回复给客户端，但是它极有可能不会回复，因为当它收到写请求时，它可能还不是TAIL。这样的话，客户端或许会重发写请求，但是这就太糟糕了，因为同一个写请求会在系统中处理两遍，所以我们需要能够在HEAD抑制重复请求。不过基本上我们讨论的所有系统都需要能够抑制重复的请求。
> 学生提问：假设第二个节点不能与HEAD进行通信，第二个节点能不能直接接管成为新的HEAD，并通知客户端将请求发给自己，而不是之前的HEAD？
> Robert教授：这是个非常好的问题。你认为呢？
> 你的方案听起来比较可行。假设HEAD和第二个节点之间的网络出问题了，



![img](https://gitee.com/zisuu/picture/raw/master/img/20210225161358.jpeg)



> HEAD还在正常运行，同时HEAD认为第二个节点挂了。然而第二个节点实际上还活着，它认为HEAD挂了。所以现在他们都会认为，另一个服务器挂了，我应该接管服务并处理写请求。因为从HEAD看来，其他服务器都失联了，HEAD会认为自己现在是唯一的副本，那么它接下来既会是HEAD，又会是TAIL。第二个节点会有类似的判断，会认为自己是新的HEAD。所以现在有了脑裂的两组数据，最终，这两组数据会变得完全不一样。

（下一节继续分析怎么解决这里的问题）

## 三 Configuration Manager

所以，Chain Replication并不能抵御网络分区，也不能抵御脑裂。在实际场景中，这意味它不能单独使用。Chain Replication是一个有用的方案，但是它不是一个完整的复制方案。它在很多场景都有使用，但是会以一种特殊的方式来使用。总是会有一个外部的权威（External Authority）来决定谁是活的，谁挂了，并确保所有参与者都认可由哪些节点组成一条链，这样在链的组成上就不会有分歧。这个外部的权威通常称为Configuration Manager。

Configuration Manager的工作就是监测节点存活性，一旦Configuration Manager认为一个节点挂了，它会生成并送出一个新的配置，在这个新的配置中，描述了链的新的定义，包含了链中所有的节点，HEAD和TAIL。Configuration Manager认为挂了的节点，或许真的挂了也或许没有，但是我们并不关心。因为所有节点都会遵从新的配置内容，所以现在不存在分歧了。

现在只有一个角色（Configuration Manager）在做决定，它不可能否认自己，所以可以解决脑裂的问题。

当然，你是如何使得一个服务是容错的，不否认自己，同时当有网络分区时不会出现脑裂呢？答案是，Configuration Manager通常会基于Raft或者Paxos。在CRAQ的场景下，它会基于Zookeeper。而Zookeeper本身又是基于类似Raft的方案。



![img](https://pic4.zhimg.com/v2-3f50a2d618ae99b0dd6a9931c1cbee0f_b.jpg)



所以，你的数据中心内的设置通常是，你有一个基于Raft或者Paxos的Configuration Manager，它是容错的，也不会受脑裂的影响。之后，通过一系列的配置更新通知，Configuration Manager将数据中心内的服务器分成多个链。比如说，Configuration Manager决定链A由服务器S1，S2，S3组成，链B由服务器S4，S5，S6组成。



![img](https://pic2.zhimg.com/v2-3766566e031ec50b4d2fc5fc936d6861_b.jpg)



Configuration Manager通告给所有参与者整个链的信息，所以所有的客户端都知道HEAD在哪，TAIL在哪，所有的服务器也知道自己在链中的前一个节点和后一个节点是什么。现在，单个服务器对于其他服务器状态的判断，完全不重要。假如第二个节点真的挂了，在收到新的配置之前，HEAD需要不停的尝试重发请求。节点自己不允许决定谁是活着的，谁挂了。

这种架构极其常见，这是正确使用Chain Replication和CRAQ的方式。在这种架构下，像Chain Replication一样的系统不用担心网络分区和脑裂，进而可以使用类似于Chain Replication的方案来构建非常高速且有效的复制系统。比如在上图中，我们可以对数据分片（Sharding），每一个分片都是一个链。其中的每一个链都可以构建成极其高效的结构来存储你的数据，进而可以同时处理大量的读写请求。同时，我们也不用太担心网络分区的问题，因为它被一个可靠的，非脑裂的Configuration Manager所管理。

> 学生提问：为什么存储具体数据的时候用Chain Replication，而不是Raft？
> Robert教授：这是一个非常合理的问题。其实数据用什么存并不重要。因为就算我们这里用了Raft，我们还是需要一个组件在产生冲突的时候来做决策。比如说数据如何在我们数百个复制系统中进行划分。如果我需要一个大的系统，我需要对数据进行分片，需要有个组件来决定数据是如何分配到不同的分区。随着时间推移，这里的划分可能会变化，因为硬件可能会有增减，数据可能会变多等等。Configuration Manager会决定以A或者B开头的key在第一个分区，以C或者D开头的key在第二个分区。至于在每一个分区，我们该使用什么样的复制方法，Chain Replication，Paxos，还是Raft，不同的人有不同的选择，有些人会使用Paxos，比如说Spanner，我们之后也会介绍。在这里，不使用Paxos或者Raft，是因为Chain Replication更加的高效，因为它减轻了Leader的负担，这或许是一个非常关键的问题。
> 某些场合可能更适合用Raft或者Paxos，因为它们不用等待一个慢的副本。而当有一个慢的副本时，Chain Replication会有性能的问题，因为每一个写请求需要经过每一个副本，只要有一个副本变慢了，就会使得所有的写请求处理变慢。这个可能非常严重，比如说你有1000个服务器，因为某人正在安装软件或者其他的原因，任意时间都有几个服务器响应比较慢。每个写请求都受限于当前最慢的服务器，这个影响还是挺大的。然而对于Raft，如果有一个副本响应速度较慢，Leader只需要等待过半服务器，而不用等待所有的副本。最终，所有的副本都能追上Leader的进度。所以，Raft在抵御短暂的慢响应方面表现的更好。一些基于Paxos的系统，也比较擅长处理副本相距较远的情况。对于Raft和Paxos，你只需要过半服务器确认，所以不用等待一个远距离数据中心的副本确认你的操作。这些原因也使得人们倾向于使用类似于Raft和Paxos这样的选举系统，而不是Chain Replication。这里的选择取决于系统的负担和系统要实现的目标。
> 不管怎样，配合一个外部的权威机构这种架构，我不确定是不是万能的，但的确是非常的通用。
> 学生提问：如果Configuration Manger认为两个服务器都活着，但是两个服务器之间的网络实际中断了会怎样？
> Robert教授：对于没有网络故障的环境，总是可以假设计算机可以通过网络互通。对于出现网络故障的环境，可能是某人踢到了网线，一些路由器被错误配置了或者任何疯狂的事情都可能发生。所以，因为错误的配置你可能陷入到这样一个情况中，Chain  Replication中的部分节点可以与Configuration Manager通信，并且Configuration Manager认为它们是活着的，但是它们彼此之间不能互相通信。



![img](https://pic1.zhimg.com/v2-00f7a5196370c37aaac1da2da087fa60_b.jpg)



> 这是这种架构所不能处理的情况。如果你希望你的系统能抵御这样的故障。你的Configuration Manager需要更加小心的设计，它需要选出不仅是它能通信的服务器，同时这些服务器之间也能相互通信。在实际中，任意两个节点都有可能网络不通。