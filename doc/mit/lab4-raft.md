# 一 脑裂



在之前的课程中，我们介绍了几个具备容错特性（fault-tolerant）的系统。如果你有留心的话，你会发现，它们有一个共同的特点。

- MapReduce复制了计算，但是复制这个动作，或者说整个MapReduce被一个单主节点控制。
- GFS以主备（primary-backup）的方式复制数据。它会实际的复制文件内容。但是它也依赖一个单主节点，来确定每一份数据的主拷贝的位置。
- VMware FT，它在一个Primary虚机和一个Backup虚机之间复制计算相关的指令。但是，当其中一个虚机出现故障时，为了能够正确的恢复。需要一个Test-and-Set服务来确认，Primary虚机和Backup虚机只有一个能接管计算任务。

这三个例子中，它们都是一个多副本系统（replication system），但是在背后，它们存在一个共性：它们需要一个单节点来决定，在多个副本中，谁是主（Primary）。

使用一个单节点的好处是，它不可能否认自己。因为只有一个节点，它的决策就是整体的决策。但是使用单节点的缺点是，它本身又是一个单点故障（Single Point of Failure）。

所以，你可以认为我们前面介绍的这些系统，它们将系统容错的关键点，转移到了这个单点上。这个单点，会在系统出现局部故障时，选择数据的主拷贝来继续工作。使用单点的原因是，我们需要避免脑裂（Split-Brain）。当出现故障时，我们之所以要极其小心的决定数据的主拷贝，是因为，如果不这么做的话，我们可能需要面临脑裂的场景。

为了让同学们更深入的了解脑裂，我接下来会说明脑裂带来的问题，以及为什么这是个严重的问题。现在，假设我们将VMware FT中的Test-and-Set服务构建成多副本的。之前这是一个单点服务，而VMware FT依赖这个Test-and-Set服务来确定Primary虚机，所以，为了提高系统的容错性，我们来构建一个多副本的Test-and-Set服务。我们来看一下，为什么出现故障时，很难避免脑裂。

现在，我们来假设我们有一个网络，这个网络里面有两个服务器（S1，S2），这两个服务器都是我们Test-and-Set服务的拷贝。这个网络里面还有两个客户端（C1，C2），它们需要通过Test-and-Set服务确定主节点是谁。在这个例子中，这两个客户端本身就是VMware FT中的Primary和Backup虚拟机。



![img](https://pic2.zhimg.com/v2-bd9d286e0fd3ab7f2752b462bdcdc585_b.jpg)



如果这是一个Test-and-Set服务，那么你知道这两个服务器中的数据记录将从0开始。任意一个客户端发送Test-and-Set指令，这个指令会将服务器中的状态设置成1。所以在这个图里面，两个服务器都应该设置成1，然后将旧的值0，返回给客户端。本质上来说，这是一种简化了的锁服务。

当一个客户端可以与其中一个服务器通信，但是不能与另一个通信时，有可能出现脑裂的问题。我们假设，客户端发送请求时，它会将请求同时发送给两个服务器。这样，我们就需要考虑，当某个服务器不响应时，客户端该怎么做？或者说，某个服务器不响应时，整个系统该如何响应？更具体点，我们假设C1可以访问S1但是不能访问S2，系统该如何响应？

一种情况是，我们必然不想让C1只与S1通信。因为，如果我们只将C1的请求设置给S1，而不设置给S2，会导致S2的数据不一致。所以，我们或许应该规定，对于任何操作，客户端必须总是与两个服务器交互，而不是只与其中一个服务器交互。但是这是一个错误的想法，为什么呢？因为这里根本就没有容错。这里甚至比只使用一个服务器更糟。因为当两个服务器中的一个故障了或者失联了，我们的系统就不能工作了。对于一个单点的服务，我们只依赖一个服务器。现在我们有两个服务器，并且两个服务器都必须一致在线，这里的难度比单个服务器更大。如果这种方式不是容错的，我们需要一种行之有效的方法。

另一个明显的答案是，如果客户端不能同时与两个服务器交互，那它就与它能连通的那个服务器交互，同时认为另一个服务器已经关机了。为什么这也是一个错误的答案呢？因为，我们的故障场景是，另一个服务器其实还开机着。我们假设我们经历的实际问题并不是这个服务器关机了，因为如果关机了对我们来说其实更好。实际情况可能更糟糕，实际可能是网络线路出现了故障，从而导致C1可以与S1交互，但是不能与S2交互。同时，C2可以与S2交互，但是不能与S1交互。现在我们规定，如果一个客户端连接了两个服务器，为了达到一定的容错性，客户端只与其中一个服务器交互也应该可以正常工作。但是这样就不可避免的出现了这种情况：假设这根线缆中断了，将网络分为两个部分。



![img](https://pic2.zhimg.com/v2-576568d9ae58e14e35abb93ddc0bf78d_b.jpg)



C1发送Test-and-Set请求给S1，S1将自己的状态设置为1，并返回之前的状态0给C1。



![img](https://pic3.zhimg.com/v2-f9ef67ca688e619c6ea9c3d3ca728c62_b.jpg)



这就意味着，C1会认为自己持有锁。如果这是一个VMware FT，C1对应的虚拟机会认为自己可以成为主节点。

但是同时，S2里面的状态仍然是0。所以如果现在C2也发送了一个Test-and-Set请求，本来应该发送给两个服务器，但是现在从C2看来，S1不能访问，根据之前定义的规则，那就发送给S2吧。同样的C2也会认为自己持有了锁。如果这个Test-and-Set服务被VMware FT使用，那么这两个VMware 虚机都会认为自己成为了主虚拟机而不需要与另一个虚拟机协商，所以这是一个错误的场景。

所以，在这种有两个拷贝副本的配置中，看起来我们只有两种选择：要么等待两个服务器响应，那么这个时候就没有容错能力；要么只等待一个服务器响应，那么就会进入错误的场景，而这种错误的场景，通常被称为脑裂。

这基本是上世纪80年代之前要面临的挑战。但是，当时又的确有多副本系统的要求。例如，控制电话交换机的计算机系统，或者是运行银行系统的计算机系统。当时的人们在构建多副本系统时，需要排除脑裂的可能。这里有两种技术：

- 第一种是构建一个不可能出现故障的网络。实际上，不可能出现故障的网络一直在我们的身边。你们电脑中，连接了CPU和内存的线路就是不可能出现故障的网络。所以，带着合理的假设和大量的资金，同时小心的控制物理环境，比如不要将一根网线拖在地上，让谁都可能踩上去。如果网络不会出现故障，这样就排除了脑裂的可能。这里做了一些假设，但是如果有足够的资金，人们可以足够接近这个假设。当网络不出现故障时，那就意味着，如果客户端不能与一个服务器交互，那么这个服务器肯定是关机了。
- 另一种就是人工解决问题，不要引入任何自动完成的操作。默认情况下，客户端总是要等待两个服务器响应，如果只有一个服务器响应，永远不要执行任何操作。相应的，给运维人员打电话，让运维人员去机房检查两个服务器。要么将一台服务器直接关机，要么确认一下其中一台服务器真的关机了，而另一个台还在工作。所以本质上，这里把人作为了一个决策器。而如果把人看成一台电脑的话，那么这个人他也是个单点。

所以，很长一段时间内，人们都使用以上两种方式中的一种来构建多副本系统。这虽然不太完美，因为人工响应不能很及时，而不出现故障的网络又很贵，但是这些方法至少是可行的。

# 二 过半投票

尽管存在脑裂的可能，但是随着技术的发展，人们发现哪怕网络可能出现故障，可能出现分区，实际上是可以正确的实现能够**自动完成故障切换**的系统。当网络出现故障，将网络分割成两半，网络的两边独自运行，且不能访问对方，这通常被称为网络分区。

![img](https://pic2.zhimg.com/v2-576568d9ae58e14e35abb93ddc0bf78d_b.jpg)


在构建能自动恢复，同时又避免脑裂的多副本系统时，人们发现，关键点在于过半票决（Majority Vote）。这是Raft论文中出现的，用来构建Raft的一个基本概念。过半票决系统的第一步在于，服务器的数量要是奇数，而不是偶数。例如在上图中（只有两个服务器），中间出现故障，那两边就太过对称了。这里被网络故障分隔的两边，它们看起来完全是一样的，它们运行了同样的软件，所以它们也会做相同的事情，这样不太好（会导致脑裂）。
但是，如果服务器的数量是奇数的，那么当出现一个网络分割时，两个网络分区将不再对称。假设出现了一个网络分割，那么一个分区会有两个服务器，另一个分区只会有一个服务器，这样就不再是对称的了。这是过半票决吸引人的地方。所以，首先你要有奇数个服务器。然后为了完成任何操作，例如Raft的Leader选举，例如提交一个Log条目，**在任何时候为了完成任何操作，你必须凑够过半的服务器来批准相应的操作**。这里的过半是指超过服务器总数的一半。直观来看，如果有3个服务器，那么需要2个服务器批准才能完成任何的操作。
这里背后的逻辑是，如果网络存在分区，那么必然不可能有超过一个分区拥有过半数量的服务器。例如，假设总共有三个服务器，如果一个网络分区有一个服务器，那么它不是一个过半的分区。如果一个网络分区有两个服务器，那么另一个分区必然只有一个服务器。因此另一个分区必然不能凑齐过半的服务器，也必然不能完成任何操作。
这里有一点需要明确，当我们在说过半的时候，我们是在说所有服务器数量的一半，而不是当前开机服务器数量的一半。这个点困扰了我（Robert教授）很长时间。如果你有一个系统有3个服务器，其中某些已经故障了，如果你要凑齐过半的服务器，你总是需要从3个服务器中凑出2个，即便你知道1个服务器已经因为故障关机了。过半总是相对于服务器的总数来说。
对于过半票决，可以用一个更通用的方程式来描述。在一个过半票决的系统中，如果有3台服务器，那么需要至少2台服务器来完成任意的操作。换个角度来看，这个系统可以接受1个服务器的故障，任意2个服务器都足以完成操作。如果你需要构建一个更加可靠的系统，那么你可以为系统加入更多的服务器。所以，更通用的方程是：
**如果系统有 2 \* F + 1 个服务器，那么系统最多可以接受F个服务器出现故障，仍然可以正常工作。**

![img](https://pic1.zhimg.com/v2-0ce76accf0a78114f8691f23f18c3e2c_b.jpg)


通常这也被称为多数投票（quorum）系统，因为3个服务器中的2个，就可以完成多数投票。
前面已经提过，有关过半票决系统的一个特性就是，最多只有一个网络分区会有过半的服务器，所以我们不可能有两个分区可以同时完成操作。这里背后更微妙的点在于，如果你总是需要过半的服务器才能完成任何操作，同时你有一系列的操作需要完成，其中的每一个操作都需要过半的服务器来批准，例如选举Raft的Leader，那么每一个操作对应的过半服务器，必然至少包含一个服务器存在于上一个操作的过半服务器中。也就是说，任意两组过半服务器，至少有一个服务器是重叠的。实际上，相比其他特性，Raft更依赖这个特性来避免脑裂。例如，当一个Raft Leader竞选成功，那么这个Leader必然凑够了过半服务器的选票，而这组过半服务器中，必然与旧Leader的过半服务器有重叠。所以，新的Leader必然知道旧Leader使用的任期号（term number），因为新Leader的过半服务器必然与旧Leader的过半服务器有重叠，而旧Leader的过半服务器中的每一个必然都知道旧Leader的任期号。类似的，任何旧Leader提交的操作，必然存在于过半的Raft服务器中，而任何新Leader的过半服务器中，必然有至少一个服务器包含了旧Leader的所有操作。这是Raft能正确运行的一个重要因素。
学生提问：可以为Raft添加服务器吗？
Rober教授：Raft的服务器是可以添加或者修改的，Raft的论文有介绍，可能在Section 6。如果是一个长期运行的系统，例如运行5年或者10年，你可能需要定期更换或者升级一些服务器，因为某些服务器可能会出现永久的故障，又或者你可能需要将服务器搬到另一个机房去。所以，肯定需要支持修改Raft服务器的集合。虽然这不是每天都发生，但是这是一个长期运行系统的重要维护工作。Raft的作者提出了方法来处理这种场景，但是比较复杂。
所以，在过半票决这种思想的支持下，大概1990年的时候，有两个系统基本同时被提出。这两个系统指出，你可以使用这种过半票决系统，从某种程度上来解决之前明显不可能避免的脑裂问题，例如，通过使用3个服务器而不是2个，同时使用过半票决策略。两个系统中的一个叫做Paxos，Raft论文对这个系统做了很多的讨论；另一个叫做ViewStamped Replication（VSR）。尽管Paxos的知名度高得多，Raft从设计上来说，与VSR更接近。VSR是由MIT发明的。这两个系统有着数十年的历史，但是他们仅仅是在15年前，也就是他们发明的15年之后，才开始走到最前线，被大量的大规模分布式系统所使用。

# 三 RAFT初探

Raft会以库（Library）的形式存在于服务中。如果你有一个基于Raft的多副本服务，那么每个服务的副本将会由两部分组成：应用程序代码和Raft库。应用程序代码接收RPC或者其他客户端请求；不同节点的Raft库之间相互合作，来维护多副本之间的操作同步。

从软件的角度来看一个Raft节点，我们可以认为在该节点的上层，是应用程序代码。例如对于Lab 3来说，这部分应用程序代码就是一个Key-Value数据库。应用程序通常都有状态，Raft层会帮助应用程序将其状态拷贝到其他副本节点。对于一个Key-Value数据库而言，对应的状态就是Key-Value Table。应用程序往下，就是Raft层。所以，Key-Value数据库需要对Raft层进行函数调用，来传递自己的状态和Raft反馈的信息。



![img](https://pic2.zhimg.com/v2-a7337077cc011215846adc3c9e40025d_b.jpg)



同时，如Raft论文中的图2所示，Raft本身也会保持状态。对我们而言，Raft的状态中，最重要的就是Raft会记录操作的日志。



![img](https://pic4.zhimg.com/v2-ad3b9b59659810de47092ed464c18ee3_b.jpg)



对于一个拥有三个副本的系统来说，很明显我们会有三个服务器，这三个服务器有完全一样的结构（上面是应用程序层，下面是Raft层）。理想情况下，也会有完全相同的数据分别存在于两层（应用程序层和Raft层）中。除此之外，还有一些客户端，假设我们有了客户端1（C1），客户端2（C2）等等。



![img](https://pic1.zhimg.com/v2-fc6ffa6ffaa14370feda57702cf2ab00_b.jpg)



客户端就是一些外部程序代码，它们想要使用服务，同时它们不知道，也没有必要知道，它们正在与一个多副本服务交互。从客户端的角度来看，这个服务与一个单点服务没有区别。

客户端会将请求发送给当前Raft集群中的Leader节点对应的应用程序。这里的请求就是应用程序级别的请求，例如一个访问Key-Value数据库的请求。这些请求可能是Put也可能是Get。Put请求带了一个Key和一个Value，将会更新Key-Value数据库中，Key对应的Value；而Get向当前服务请求某个Key对应的Value。



![img](https://pic1.zhimg.com/v2-57e0d1b2035621b2904d4e20a2e922f4_b.jpg)



所以，看起来似乎没有Raft什么事，看起来就像是普通的客户端服务端交互。一旦一个Put请求从客户端发送到了服务端，对于一个单节点的服务来说，应用程序会直接执行这个请求，更新Key-Value表，之后返回对于这个Put请求的响应。但是对于一个基于Raft的多副本服务，就要复杂一些。

假设客户端将请求发送给Raft的Leader节点，在服务端程序的内部，应用程序只会将来自客户端的请求对应的操作向下发送到Raft层，并且告知Raft层，请把这个操作提交到多副本的日志（Log）中，并在完成时通知我。



![img](https://pic2.zhimg.com/v2-4cb46ee22091d6d343739ae8e50b9f51_b.jpg)



之后，Raft节点之间相互交互，直到过半的Raft节点将这个新的操作加入到它们的日志中，也就是说这个操作被过半的Raft节点复制了。



![img](https://pic2.zhimg.com/v2-6ad90c4fbc46242686ba2bf6a433520d_b.jpg)



当且仅当Raft的Leader节点知道了所有（课程里说的是所有，但是这里应该是过半节点）的副本都有了这个操作的拷贝之后。Raft的Leader节点中的Raft层，会向上发送一个通知到应用程序，也就是Key-Value数据库，来说明：刚刚你提交给我的操作，我已经提交给所有（注：同上一个说明）副本，并且已经成功拷贝给它们了，现在，你可以真正的执行这个操作了。



![img](https://pic4.zhimg.com/v2-1a64849a86d69b2209d8c2d615378793_b.jpg)



所以，客户端发送请求给Key-Value数据库，这个请求不会立即被执行，因为这个请求还没有被拷贝。当且仅当这个请求存在于过半的副本节点中时，Raft才会通知Leader节点，只有在这个时候，Leader才会实际的执行这个请求。对于Put请求来说，就是更新Value，对于Get请求来说，就是读取Value。最终，请求返回给客户端，这就是一个普通请求的处理过程。

> 学生提问：问题听不清。。。这里应该是学生在纠正前面对于所有节点和过半节点的混淆
> Robert教授：这里只需要拷贝到过半服务器即可。为什么不需要拷贝到所有的节点？因为我们想构建一个容错系统，所以即使某些服务器故障了，我们依然期望服务能够继续工作。所以只要过半服务器有了相应的拷贝，那么请求就可以提交。
> 学生提问：除了Leader节点，其他节点的应用程序层会有什么样的动作？
> Robert教授：哦对，抱歉。当一个操作最终在Leader节点被提交之后，每个副本节点的Raft层会将相同的操作提交到本地的应用程序层。在本地的应用程序层，会将这个操作更新到自己的状态。所以，理想情况是，所有的副本都将看到相同的操作序列，这些操作序列以相同的顺序出现在Raft到应用程序的upcall中，之后它们以相同的顺序被本地应用程序应用到本地的状态中。假设操作是确定的（比如一个随机数生成操作就不是确定的），所有副本节点的状态，最终将会是完全一样的。我们图中的Key-Value数据库，就是Raft论文中说的状态（也就是Key-Value数据库的多个副本最终会保持一致）。



![img](https://pic1.zhimg.com/v2-c8f62c0ebd2967b31c7f9c67bb7d0d3c_b.jpg)

# 四 LOG同步时序

这一部分我们从另一个角度来看Raft Log同步的一些交互，这种角度将会在这门课中出现很多次，那就是时序图。

接下来我将画一个时序图来描述Raft内部的消息是如何工作的。假设我们有一个客户端，服务器1是当前Raft集群的Leader。同时，我们还有服务器2，服务器3。这张图的纵坐标是时间，越往下时间越长。假设客户端将请求发送给服务器1，这里的客户端请求就是一个简单的请求，例如一个Put请求。



![img](https://pic3.zhimg.com/v2-1aa81770feb3a23886e60df9c3d4743e_b.jpg)



之后，服务器1的Raft层会发送一个添加日志（AppendEntries）的RPC到其他两个副本（S2，S3）。现在服务器1会一直等待其他副本节点的响应，一直等到过半节点的响应返回。这里的过半节点包括Leader自己。所以在一个只有3个副本节点的系统中，Leader只需要等待一个其他副本节点。



![img](https://pic4.zhimg.com/v2-e4edec7e8db19609cf55d5a5cc3e44bb_b.jpg)



一旦过半的节点返回了响应，这里的过半节点包括了Leader自己，所以在一个只有3个副本的系统中，Leader只需要等待一个其他副本节点返回对于AppendEntries的正确响应。



![img](https://pic3.zhimg.com/v2-17d92431a8e08d17c7956faa4e993c1e_b.jpg)



当Leader收到了过半服务器的正确响应，Leader会执行（来自客户端的）请求，得到结果，并将结果返回给客户端。



![img](https://pic1.zhimg.com/v2-b3209eaeda33cf4a01a57bb7e349bc98_b.jpg)



与此同时，服务器3可能也会将它的响应返回给Leader，尽管这个响应是有用的，但是这里不需要等待这个响应。这一点对于理解Raft论文中的图2是有用的。



![img](https://pic4.zhimg.com/v2-f20e8c7003f6b60acf4e8ab1e50a7d3b_b.jpg)



好了，大家明白了吗？这是系统在没有故障情况下，处理普通操作的流程。

> 学生提问：S2和S3的状态怎么保持与S1同步？
> Robert教授：我的天，我忘了一些重要的步骤。现在Leader知道过半服务器已经添加了Log，可以执行客户端请求，并返回给客户端。但是服务器2还不知道这一点，服务器2只知道：我从Leader那收到了这个请求，但是我不知道这个请求是不是已经被Leader提交（committed）了，这取决于我的响应是否被Leader收到。服务器2只知道，它的响应提交给了网络，或许Leader没有收到这个响应，也就不会决定commit这个请求。所以这里还有一个阶段。一旦Leader发现请求被commit之后，它需要将这个消息通知给其他的副本。所以这里有一个额外的消息。



![img](https://pic3.zhimg.com/v2-a772153d24e4d2e02eca9d4595eb1e5e_b.jpg)



这条消息的具体内容依赖于整个系统的状态。至少在Raft中，没有明确的committed消息。相应的，committed消息被夹带在下一个AppendEntries消息中，由Leader下一次的AppendEntries对应的RPC发出。任何情况下，当有了committed消息时，这条消息会填在AppendEntries的RPC中。下一次Leader需要发送心跳，或者是收到了一个新的客户端请求，要将这个请求同步给其他副本时，Leader会将新的更大的commit号随着AppendEntries消息发出，当其他副本收到了这个消息，就知道之前的commit号已经被Leader提交，其他副本接下来也会执行相应的请求，更新本地的状态。



![img](https://pic1.zhimg.com/v2-3a8fc7271c6df97636df0e5bb8cf7294_b.jpg)



> 学生提问：这里的内部交互有点多吧？
> Robert教授：是的，这是一个内部需要一些交互的协议，它不是特别的快。实际上，客户端发出请求，请求到达某个服务器，这个服务器至少需要与一个其他副本交互，在返回给客户端之前，需要等待多条消息。所以，一个客户端响应的背后有多条消息的交互。
> 学生提问：也就是说commit信息是随着普通的AppendEntries消息发出的？那其他副本的状态更新就不是很及时了。
> Robert教授：是的，作为实现者，这取决于你在什么时候将新的commit号发出。如果客户端请求很稀疏，那么Leader或许要发送一个心跳或者发送一条特殊的AppendEntries消息。如果客户端请求很频繁，那就无所谓了。因为如果每秒有1000个请求，那么下一条AppendEntries很快就会发出，你可以在下一条消息中带上新的commit号，而不用生成一条额外的消息。额外的消息代价还是有点高的，反正你要发送别的消息，可以把新的commit号带在别的消息里。
> 实际上，我不认为其他副本（非Leader）执行客户端请求的时间很重要，因为没有人在等这个步骤。至少在不出错的时候，其他副本执行请求是个不太重要的步骤。例如说，客户端就没有等待其他副本执行请求，客户端只会等待Leader执行请求。所以，其他副本在什么时候执行请求，不会影响客户端感受的请求时延。6.4 Log 同步时序

# 五 日志

你们应该关心的一个问题是：为什么Raft系统这么关注Log，Log究竟起了什么作用？这个问题值得好好来回答一下。

Raft系统之所以对Log关注这么多的一个原因是，Log是Leader用来对操作排序的一种手段。这对于复制状态机（详见4.2）而言至关重要，对于这些复制状态机来说，所有副本不仅要执行相同的操作，还需要用相同的顺序执行这些操作。而Log与其他很多事物，共同构成了Leader对接收到的客户端操作分配顺序的机制。比如说，我有10个客户端同时向Leader发出请求，Leader必须对这些请求确定一个顺序，并确保所有其他的副本都遵从这个顺序。实际上，Log是一些按照数字编号的槽位（类似一个数组），槽位的数字表示了Leader选择的顺序。

Log的另一个用途是，在一个（非Leader，也就是Follower）副本收到了操作，但是还没有执行操作时。该副本需要将这个操作存放在某处，直到收到了Leader发送的新的commit号才执行。所以，对于Raft的Follower来说，Log是用来存放临时操作的地方。Follower收到了这些临时的操作，但是还不确定这些操作是否被commit了。我们将会看到，这些操作可能会被丢弃。

Log的另一个用途是用在Leader节点，我（Robert教授）很喜欢这个特性。Leader需要在它的Log中记录操作，因为这些操作可能需要重传给Follower。如果一些Follower由于网络原因或者其他原因短时间离线了或者丢了一些消息，Leader需要能够向Follower重传丢失的Log消息。所以，Leader也需要一个地方来存放客户端请求的拷贝。即使对那些已经commit的请求，为了能够向丢失了相应操作的副本重传，也需要存储在Leader的Log中。

所有节点都需要保存Log还有一个原因，就是它可以帮助重启的服务器恢复状态。你可能的确需要一个故障了的服务器在修复后，能重新加入到Raft集群，要不然你就永远少了一个服务器。比如对于一个3节点的集群来说，如果一个节点故障重启之后不能自动加入，那么当前系统只剩2个节点，那将不能再承受任何故障，所以我们需要能够重新并入故障重启了的服务器。对于一个重启的服务器来说，会使用存储在磁盘中的Log。每个Raft节点都需要将Log写入到它的磁盘中，这样它故障重启之后，Log还能保留。而这个Log会被Raft节点用来从头执行其中的操作进而重建故障前的状态，并继续以这个状态运行。所以，Log也会被用来持久化存储操作，服务器可以依赖这些操作来恢复状态。

> 学生提问：假设Leader每秒可以执行1000条操作，Follower只能每秒执行100条操作，并且这个状态一直持续下去，会怎样？
> Robert（教授）：这里有一点需要注意，Follower在实际执行操作前会确认操作。所以，它们会确认，并将操作堆积在Log中。而Log又是无限的，所以Follower或许可以每秒确认1000个操作。如果Follower一直这么做，它会生成无限大的Log，因为Follower的执行最终将无限落后于Log的堆积。 所以，当Follower堆积了10亿（不是具体的数字，指很多很多）Log未执行，最终这里会耗尽内存。之后Follower调用内存分配器为Log申请新的内存时，内存申请会失败。Raft并没有流控机制来处理这种情况。
> 所以我认为，在一个实际的系统中，你需要一个额外的消息，这个额外的消息可以夹带在其他消息中，也不必是实时的，但是你或许需要一些通信来（让Follower）告诉Leader，Follower目前执行到了哪一步。这样Leader就能知道自己在操作执行上领先太多。所以是的，我认为在一个生产环境中，如果你想使用系统的极限性能，你还是需要一条额外的消息来调节Leader的速度。



![img](https://gitee.com/zisuu/picture/raw/master/img/20210221170420.jpeg)



> 学生提问：如果其中一个服务器故障了，它的磁盘中会存有Log，因为这是Raft论文中图2要求的，所以服务器可以从磁盘中的Log恢复状态，但是这个服务器不知道它当前在Log中的执行位置。同时，当它第一次启动时，它也不知道那些Log被commit了。
> Robert教授：所以，对于第一个问题的答案是，一个服务器故障重启之后，它会立即读取Log，但是接下来它不会根据Log做任何操作，因为它不知道当前的Raft系统对Log提交到了哪一步，或许有1000条未提交的Log。
> 学生补充问题：如果Leader出现了故障会怎样？
> Robert教授：如果Leader也关机也没有区别。让我们来假设Leader和Follower同时故障了，那么根据Raft论文图2，它们只有non-volatile状态（也就是磁盘中存储的状态）。这里的状态包括了Log和最近一次任期号（Term Number）。如果大家都出现了故障然后大家都重启了，它们中没有一个在刚启动的时候就知道它们在故障前执行到了哪一步。所以这个时候，会先进行Leader选举，其中一个被选为Leader。如果你回顾一下Raft论文中的图2有关AppendEntries的描述，这个Leader会在发送第一次心跳时弄清楚，整个系统中目前执行到了哪一步。Leader会确认一个过半服务器认可的最近的Log执行点，这就是整个系统的执行位置。另一种方式来看这个问题，一旦你通过AppendEntries选择了一个Leader，这个Leader会迫使其他所有副本的Log与自己保持一致。这时，再配合Raft论文中介绍的一些其他内容，由于Leader知道它迫使其他所有的副本都拥有与自己一样的Log，那么它知道，这些Log必然已经commit，因为它们被过半的副本持有。这时，按照Raft论文的图2中对AppendEntries的描述，Leader会增加commit号。之后，所有节点可以从头开始执行整个Log，并从头构造自己的状态。但是这里的计算量或许会非常大。所以这是Raft论文的图2所描述的过程，很明显，这种从头开始执行的机制不是很好，但是这是Raft协议的工作流程。下一课我们会看一种更有效的，利用checkpoint的方式。

所以，这就是普通的，无故障操作的时序。

# 六 应用层接口



这一部分简单介绍一下应用层和Raft层之间的接口。你或许已经通过实验了解了一些，但是我们这里大概来看一下。假设我们的应用程序是一个key-value数据库，下面一层是Raft层。



![img](https://pic2.zhimg.com/v2-b876668369b397da4c93b4c495a53011_b.jpg)



在Raft集群中，每一个副本上，这两层之间主要有两个接口。

第一个接口是key-value层用来转发客户端请求的接口。如果客户端发送一个请求给key-value层，key-value层会将这个请求转发给Raft层，并说：请将这个请求存放在Log中的某处。



![img](https://pic1.zhimg.com/v2-4137e0dcc817c140a57bb7695bd86f50_b.jpg)



这个接口实际上是个函数调用，称之为Start函数。这个函数只接收一个参数，就是客户端请求。key-value层说：我接到了这个请求，请把它存在Log中，并在committed之后告诉我。



![img](https://pic4.zhimg.com/v2-6cfac3568f30dbb6ef54d294f77e80f7_b.jpg)



另一个接口是，随着时间的推移，Raft层会通知key-value层：哈，你刚刚在Start函数中传给我的请求已经commit了。Raft层通知的，不一定是最近一次Start函数传入的请求。例如在任何请求commit之前，可能会再有超过100个请求通过Start函数传给Raft层。



![img](https://pic4.zhimg.com/v2-8d123253e6da9c73c8efec14c2dd8583_b.jpg)



这个向上的接口以go channel中的一条消息的形式存在。Raft层会发出这个消息，key-value层要读取这个消息。所以这里有个叫做applyCh的channel，通过它你可以发送ApplyMsg消息。



![img](https://pic2.zhimg.com/v2-5a87ab81535c7b4717fb56436be3686d_b.jpg)



当然，key-value层需要知道从applyCh中读取的消息，对应之前调用的哪个Start函数，所以Start函数的返回需要有足够的信息给key-value层，这样才能完成对应。Start函数的返回值包括，这个请求将会存放在Log中的位置（index）。这个请求不一定能commit成功，但是如果commit成功的话，会存放在这个Log位置。同时，它还会返回当前的任期号（term number）和一些其它我们现在还不太关心的内容。



![img](https://pic3.zhimg.com/v2-945fbeea2156b9206a65caf3667facf2_b.jpg)



在ApplyMsg中，将会包含请求（command）和对应的Log位置（index）。



![img](https://pic1.zhimg.com/v2-966438ecd5aff1972914abee6176c3a8_b.jpg)



所有的副本都会收到这个ApplyMsg消息，它们都知道自己应该执行这个请求，弄清楚这个请求的具体含义，并将它应用在本地的状态中。所有的副本节点还会拿到Log的位置信息（index），但是这个位置信息只在Leader有用，因为Leader需要知道ApplyMsg中的请求究竟对应哪个客户端请求（进而响应客户端请求）。

> 学生提问：为什么不在Start函数返回的时候就响应客户端请求呢？
> Robert教授：我们假设客户端发送了任意的请求，我们假设这里是一个Put或者Get请求，是什么其实不重要，我们还是假设这里是个Get请求。客户端发送了一个Get请求，并且等待响应。当Leader知道这个请求被（Raft）commit之后，会返回响应给客户端。所以这里会是一个Get响应。所以，（在Leader返回响应之前）客户端看不到任何内容。
> 这意味着，在实际的软件中，客户端调用key-value的RPC，key-value层收到RPC之后，会调用Start函数，Start函数会立即返回，但是这时，key-value层不会返回消息给客户端，因为它还没有执行客户端请求，它也不知道这个请求是否会被（Raft）commit。一个不能commit的场景是，当key-value层调用了Start函数，Start函数返回之后，它就故障了，所以它必然没有发送Apply Entry消息或者其他任何消息，所以也不能执行commit。
> 所以实际上，Start函数返回了，随着时间的推移，对应于这个客户端请求的ApplyMsg从applyCh channel中出现在了key-value层。只有在那个时候，key-value层才会执行这个请求，并返回响应给客户端。

有一件事情你们需要熟悉，那就是，首先，对于Log来说有一件有意思的事情：不同副本的Log或许不完全一样。有很多场合都会不一样，至少不同副本节点的Log的末尾，会短暂的不同。例如，一个Leader开始发出一轮AppendEntries消息，但是在完全发完之前就故障了。这意味着某些副本收到了这个AppendEntries，并将这条新Log存在本地。而那些没有收到AppendEntries消息的副本，自然也不会将这条新Log存入本地。所以，这里很容易可以看出，不同副本中，Log有时会不一样。

不过对于Raft来说，Raft会最终强制不同副本的Log保持一致。或许会有短暂的不一致，但是长期来看，所有副本的Log会被Leader修改，直到Leader确认它们都是一致的。

接下来会有有关Raft的两个大的主题，一个是Lab2的内容：Leader Election是如何工作的；另一个是，Leader如何处理不同的副本日志的差异，尤其在出现故障之后。

# 七 Leader选取



这一部分我们来看一下Leader选举。这里有个问题，为什么Raft系统会有个Leader，为什么我们需要一个Leader？

答案是，你可以不用Leader就构建一个类似的系统。实际上有可能不引入任何指定的Leader，通过一组服务器来共同认可Log的顺序，进而构建一个一致系统。实际上，Raft论文中引用的Paxos系统就没有Leader，所以这是有可能的。

有很多原因导致了Raft系统有一个Leader，其中一个最主要的是：通常情况下，如果服务器不出现故障，有一个Leader的存在，会使得整个系统更加高效。因为有了一个大家都知道的指定的Leader，对于一个请求，你可以只通过一轮消息就获得过半服务器的认可。对于一个无Leader的系统，通常需要一轮消息来确认一个临时的Leader，之后第二轮消息才能确认请求。所以，使用一个Leader可以提升系统性能至2倍。同时，有一个Leader可以更好的理解Raft系统是如何工作的。

Raft生命周期中可能会有不同的Leader，它使用任期号（term number）来区分不同的Leader。Followers（非Leader副本节点）不需要知道Leader的ID，它们只需要知道当前的任期号。每一个任期最多有一个Leader，这是一个很关键的特性。对于每个任期来说，或许没有Leader，或许有一个Leader，但是不可能有两个Leader出现在同一个任期中。每个任期必然最多只有一个Leader。

那Leader是如何创建出来的呢？每个Raft节点都有一个选举定时器（Election Timer），如果在这个定时器时间耗尽之前，当前节点没有收到任何当前Leader的消息，这个节点会认为Leader已经下线，并开始一次选举。所以我们这里有了这个选举定时器，当它的时间耗尽时，当前节点会开始一次选举。



![img](https://pic4.zhimg.com/v2-397eddef4610e08bb545027087eb411b_b.jpg)



开始一次选举的意思是，当前服务器会增加任期号（term number），因为它想成为一个新的Leader。而你知道的，一个任期内不能有超过一个Leader，所以为了成为一个新的Leader，这里需要开启一个新的任期。 之后，当前服务器会发出请求投票（RequestVote）RPC，这个消息会发给所有的Raft节点。其实只需要发送到N-1个节点，因为Raft规定了，Leader的候选人总是会在选举时投票给自己。



![img](https://pic3.zhimg.com/v2-efc69ebec38400ff4401170134a6cc0a_b.jpg)



这里需要注意的一点是，并不是说如果Leader没有故障，就不会有选举。但是如果Leader的确出现了故障，那么一定会有新的选举。这个选举的前提是其他服务器还在运行，因为选举需要其他服务器的选举定时器超时了才会触发。另一方面，如果Leader没有故障，我们仍然有可能会有一次新的选举。比如，如果网络很慢，丢了几个心跳，或者其他原因，这时，尽管Leader还在健康运行，我们可能会有某个选举定时器超时了，进而开启一次新的选举。在考虑正确性的时候，我们需要记住这点。所以这意味着，如果有一场新的选举，有可能之前的Leader仍然在运行，并认为自己还是Leader。例如，当出现网络分区时，旧Leader始终在一个小的分区中运行，而较大的分区会进行新的选举，最终成功选出一个新的Leader。这一切，旧的Leader完全不知道。所以我们也需要关心，在不知道有新的选举时，旧的Leader会有什么样的行为？

（注：下面这一段实际在Lec 06的65-67分钟出现，与这一篇前后的内容在时间上不连续，但是因为内容相关就放到这里来了）

假设网线故障了，旧的Leader在一个网络分区中，这个网络分区中有一些客户端和少数（未过半）的服务器。在网络的另一个分区中，有着过半的服务器，这些服务器选出了一个新的Leader。旧的Leader会怎样，或者说为什么旧的Leader不会执行错误的操作？这里看起来有两个潜在的问题。第一个问题是，如果一个Leader在一个网络分区中，并且这个网络分区没有过半的服务器。那么下次客户端发送请求时，这个在少数分区的Leader，它会发出AppendEntries消息。但是因为它在少数分区，即使包括它自己，它也凑不齐过半服务器，所以它永远不会commit这个客户端请求，它永远不会执行这个请求，它也永远不会响应客户端，并告诉客户端它已经执行了这个请求。所以，如果一个旧的Leader在一个不同的网络分区中，客户端或许会发送一个请求给这个旧的Leader，但是客户端永远也不能从这个Leader获得响应。所以没有客户端会认为这个旧的Leader执行了任何操作。另一个更奇怪的问题是，有可能Leader在向一部分Followers发完AppendEntries消息之后就故障了，所以这个Leader还没决定commit这个请求。这是一个非常有趣的问题，我将会再花45分钟（下一节课）来讲。

> 学生提问：有没有可能出现极端的情况，导致单向的网络出现故障，进而使得Raft系统不能工作？
> Robert教授：我认为是有可能的。例如，如果当前Leader的网络单边出现故障，Leader可以发出心跳，但是又不能收到任何客户端请求。它发出的心跳被送达了，因为它的出方向网络是正常的，那么它的心跳会抑制其他服务器开始一次新的选举。但是它的入方向网络是故障的，这会阻止它接收或者执行任何客户端请求。这个场景是Raft并没有考虑的众多极端的网络故障场景之一。
> 我认为这个问题是可修复的。我们可以通过一个双向的心跳来解决这里的问题。在这个双向的心跳中，Leader发出心跳，但是这时Followers需要以某种形式响应这个心跳。如果Leader一段时间没有收到自己发出心跳的响应，Leader会决定卸任，这样我认为可以解决这个特定的问题和一些其他的问题。
> 你是对的，网络中可能发生非常奇怪的事情，而Raft协议没有考虑到这些场景。

所以，我们这里有Leader选举，我们需要确保每个任期最多只有一个Leader。Raft是如何做到这一点的呢？

为了能够当选，Raft要求一个候选人从过半服务器中获得认可投票。每个Raft节点，只会在一个任期内投出一个认可选票。这意味着，在任意一个任期内，每一个节点只会对一个候选人投一次票。这样，就不可能有两个候选人同时获得过半的选票，因为每个节点只会投票一次。所以这里是过半原则导致了最多只能有一个胜出的候选人，这样我们在每个任期会有最多一个选举出的候选人。

同时，也是非常重要的一点，过半原则意味着，即使一些节点已经故障了，你仍然可以赢得选举。如果少数服务器故障了或者出现了网络问题，我们仍然可以选举出Leader。如果超过一半的节点故障了，不可用了，或者在另一个网络分区，那么系统会不断地额尝试选举Leader，并永远也不能选出一个Leader，因为没有过半的服务器在运行。

如果一次选举成功了，整个集群的节点是如何知道的呢？当一个服务器赢得了一次选举，这个服务器会收到过半的认可投票，这个服务器会直接知道自己是新的Leader，因为它收到了过半的投票。但是其他的服务器并不能直接知道谁赢得了选举，其他服务器甚至都不知道是否有人赢得了选举。这时，（赢得了选举的）候选人，会通过心跳通知其他服务器。Raft论文的图2规定了，如果你赢得了选举，你需要立刻发送一条AppendEntries消息给其他所有的服务器。这条代表心跳的AppendEntries并不会直接说：我赢得了选举，我就是任期23的Leader。这里的表达会更隐晦一些。Raft规定，除非是当前任期的Leader，没人可以发出AppendEntries消息。所以假设我是一个服务器，我发现对于任期19有一次选举，过了一会我收到了一条AppendEntries消息，这个消息的任期号就是19。那么这条消息告诉我，我不知道的某个节点赢得了任期19的选举。所以，其他服务器通过接收特定任期号的AppendEntries来知道，选举成功了。

# 八选举定时器

选举定时器在上一篇有过一些介绍）

任何一条AppendEntries消息都会重置所有Raft节点的选举定时器。这样，只要Leader还在线，并且它还在以合理的速率（不能太慢）发出心跳或者其他的AppendEntries消息，Followers收到了AppendEntries消息，会重置自己的选举定时器，这样Leader就可以阻止任何其他节点成为一个候选人。所以只要所有环节都在正常工作，不断重复的心跳会阻止任何新的选举发生。当然，如果网络故障或者发生了丢包，不可避免的还是会有新的选举。但是如果一切都正常，我们不太可能会有一次新的选举。

如果一次选举选出了0个Leader，这次选举就失败了。有一些显而易见的场景会导致选举失败，例如太多的服务器关机或者不可用了，或者网络连接出现故障。这些场景会导致你不能凑齐过半的服务器，进而也不能赢得选举，这时什么事也不会发生。

一个导致选举失败的更有趣的场景是，所有环节都在正常工作，没有故障，没有丢包，但是候选人们几乎是同时参加竞选，它们分割了选票（Split Vote）。假设我们有一个3节点的多副本系统，3个节点的选举定时器几乎同超时，进而期触发选举。首先，每个节点都会为自己投票。之后，每个节点都会收到其他节点的RequestVote消息，因为该节点已经投票给自己了，所以它会返回反对投票。这意味着，3个节点中的每个节点都只能收到一张投票（来自于自己）。没有一个节点获得了过半投票，所以也就没有人能被选上。接下来它们的选举定时器会重新计时，因为选举定时器只会在收到了AppendEntries消息时重置，但是由于没有Leader，所有也就没有AppendEntries消息。所有的选举定时器重新开始计时，如果我们不够幸运的话，所有的定时器又会在同一时间到期，所有节点又会投票给自己，又没有人获得了过半投票，这个状态可能会一直持续下去。

Raft不能完全避免分割选票（Split Vote），但是可以使得这个场景出现的概率大大降低。Raft通过为选举定时器随机的选择超时时间来达到这一点。我们可以这样来看这种随机的方法。假设这里有个时间线，我会在上面画上事件。在某个时间，所有的节点收到了最后一条AppendEntries消息。之后，Leader就故障了。我们这里假设Leader在发出最后一次心跳之后就故障关机了。所有的Followers在同一时间重置了它们的选举定时器，因为它们大概率在同一时间收到了这条AppendEntries消息。



![img](https://pic4.zhimg.com/80/v2-f0d1a0df7fe3b33c4269c218a64f4913_720w.jpg)



它们都重置了自己的选举定时器，这样在将来的某个时间会触发选举。但是这时，它们为选举定时器选择了不同的超时时间。

假设故障的旧的Leader是服务器1，那么服务器2（S2），服务器3（S3）会在这个点为它们的选举定时器设置随机的超时时间。



![img](https://pic2.zhimg.com/80/v2-740e95a2589d23d0b65d8055b1cc56b9_720w.jpg)



假设S2的选举定时器的超时时间在这，而S3的在这。



![img](https://pic4.zhimg.com/80/v2-7433cf0c7d2f706ad17ce4737680f2b7_720w.jpg)



这个图里的关键点在于，因为不同的服务器都选取了随机的超时时间，总会有一个选举定时器先超时，而另一个后超时。假设S2和S3之间的差距足够大，先超时的那个节点（也就是S2）能够在另一个节点（也就是S3）超时之前，发起一轮选举，并获得过半的选票，那么那个节点（也就是S2）就可以成为新的Leader。大家都明白了随机化是如何去除节点之间的同步特性吗？

这里对于选举定时器的超时时间的设置，需要注意一些细节。一个明显的要求是，选举定时器的超时时间需要至少大于Leader的心跳间隔。这里非常明显，假设Leader每100毫秒发出一个心跳，你最好确认所有节点的选举定时器的超时时间不要小于100毫秒，否则该节点会在收到正常的心跳之前触发选举。所以，选举定时器的超时时间下限是一个心跳的间隔。实际上由于网络可能丢包，这里你或许希望将下限设置为多个心跳间隔。所以如果心跳间隔是100毫秒，你或许想要将选举定时器的最短超时时间设置为300毫秒，也就是3次心跳的间隔。所以，如果心跳间隔是这么多（两个AE之间），那么你会想要将选举定时器的超时时间下限设置成心跳间隔的几倍，在这里。



![img](https://pic2.zhimg.com/80/v2-603792a3651940cc5d7a098f543255d5_720w.jpg)



那超时时间的上限呢？因为随机的话都是在一个范围内随机，那我们应该在哪设置超时时间的上限呢？在一个实际系统中，有几点需要注意。



![img](https://pic2.zhimg.com/80/v2-9eb69a4528e918b4694828aaee4906b1_720w.jpg)



首先，这里的最大超时时间影响了系统能多快从故障中恢复。因为从旧的Leader故障开始，到新的选举开始这段时间，整个系统是瘫痪了。尽管还有一些其他服务器在运行，但是因为没有Leader，客户端请求会被丢弃。所以，这里的上限越大，系统的恢复时间也就越长。这里究竟有多重要，取决于我们需要达到多高的性能，以及故障出现的频率。如果一年才出一次故障，那就无所谓了。如果故障很频繁，那么我们或许就该关心恢复时间有多长。这是一个需要考虑的点。

另一个需要考虑的点是，不同节点的选举定时器的超时时间差（S2和S3之间）必须要足够长，使得第一个开始选举的节点能够完成一轮选举。这里至少需要大于发送一条RPC所需要的往返（Round-Trip）时间。



![img](https://pic2.zhimg.com/80/v2-022a9d033a88b1c6c8d5d99315d4f7f9_720w.jpg)



或许需要10毫秒来发送一条RPC，并从其他所有服务器获得响应。如果这样的话，我们需要设置超时时间的上限到足够大，从而使得两个随机数之间的时间差极有可能大于10毫秒。

在Lab2中，如果你的代码不能在几秒内从一个Leader故障的场景中恢复的话，测试代码会报错。所以这种场景下，你们需要调小选举定时器超时时间的上限。这样的话，你才可能在几秒内完成一次Leader选举。这并不是一个很严格的限制。

这里还有一个小点需要注意，每一次一个节点重置自己的选举定时器时，都需要重新选择一个随机的超时时间。也就是说，不要在服务器启动的时候选择一个随机的超时时间，然后反复使用同一个值。因为如果你不够幸运的话，两个服务器会以极小的概率选择相同的随机超时时间，那么你会永远处于分割选票的场景中。所以你需要每次都为选举定时器选择一个不同的随机超时时间。

# 九 可能出现的问题



一个旧Leader在各种奇怪的场景下故障之后，为了恢复系统的一致性，一个新任的Leader如何能整理在不同副本上可能已经不一致的Log？

这个话题只在Leader故障之后才有意义，如果Leader正常运行，Raft不太会出现问题。如果Leader正在运行，并且在其运行时，系统中有过半服务器。Leader只需要告诉Followers，Log该是什么样子。Raft要求Followers必须同意并接收Leader的Log，这在Raft论文的图2中有说明。只要Followers还能处理，它们就会全盘接收Leader在AppendEntries中发送给它们的内容，并加到本地的Log中。之后再收到来自Leader的commit消息，在本地执行请求。这里很难出错。

在Raft中，当Leader故障了才有可能出错。例如，旧的Leader在发送消息的过程中故障了，或者新Leader在刚刚当选之后，还没来得及做任何操作就故障了。所以这里有一件事情我们非常感兴趣，那就是在一系列故障之后，Log会是怎样？

这里有个例子，假设我们有3个服务器（S1，S2，S3），我将写出每个服务器的Log，每一列对齐之后就是Log的一个槽位。我这里写的值是Log条目对应的任期号，而不是Log记录的客户端请求。所以第一列是槽位1，第二列是槽位2。所有节点在任期3的时候记录了一个请求在槽位1，S2和S3在任期3的时候记录了一个请求在槽位2。在槽位2，S1没有任何记录。 



![img](https://pic3.zhimg.com/v2-159a6820a96a27da2de27781bd6418b2_b.jpg)



所以，这里的问题是：这种情况可能发生吗？如果可能发生，是怎么发生的？

这种情况是可能发生的。假设S3是任期3的Leader，它收到了一个客户端请求，之后发送给其他服务器。其他服务器收到了相应的AppendEntries消息，并添加Log到本地，这是槽位1的情况。之后，S3从客户端收到了第二个请求，它还是需要将这个请求发送给其他服务器。但是这里有三种情况：

- 发送给S1的消息丢了
- S1当时已经关机了
- S3在向S2发送完AppendEntries之后，在向S1发送AppendEntries之前故障了

现在，只有S2和S3有槽位2的Log。Leader在发送AppendEntries消息之前，总是会将新的请求加到自己的Log中（所以S3有Log），而现在AppendEntries RPC只送到了S2（所以S2有Log）。这是不同节点之间Log不一样的一种最简单的场景。我们现在知道了它是如何发生的。

如果现任Leader S3故障了，首先我们需要新的选举，之后某个节点会被选为新的Leader。接下来会发生两件事情：

- 新的Leader需要认识到，槽位2的请求可能已经commit了，从而不能丢弃。
- 新的Leader需要确保S1在槽位2记录与其他节点完全一样的请求。

这里还有另外一个例子需要考虑。还是3个服务器，这次我会给Log的槽位加上数字，这样更方便我们后面说明。我们这里有槽位10、11、12、13。槽位10和槽位11类似于前一个例子。在槽位12，S2有一个任期4的请求，而S3有一个任期5的请求。在我们分析之前，我们需要明白，发生了什么会导致这个场景？我们需要清楚这个场景是否真的存在，因为有些场景不可能存在我们也就没必要考虑它。所以现在的问题是，这种场景可能发生吗？



![img](https://pic3.zhimg.com/v2-73370661d0ffadaf6ede509974ce55fa_b.jpg)



这种场景是可能发生的。我们假设S2在槽位12时，是任期4的新Leader，它收到了来自客户端的请求，将这个请求加到了自己的Log中，然后就故障了。



![img](https://pic1.zhimg.com/v2-bddd3614a5471c978d6e2a921c95f430_b.jpg)



因为Leader故障了，我们需要一次新的选举。我们来看哪个服务器可以被选为新的Leader。这里S3可能被选上，因为它只需要从过半服务器获得认可投票，而在这个场景下，过半服务器就是S1和S3。所以S3可能被选为任期5的新Leader，之后收到了来自客户端的请求，将这个请求加到自己的Log中，然后故障了。之后就到了例子中的场景了。



![img](https://pic4.zhimg.com/v2-a46063c14caaa61fc6fa54b63c888747_b.jpg)



因为可能发生，Raft必须能够处理这种场景。在我们讨论Raft会如何做之前，我们必须了解，怎样才是一种可接受的结果。大概看一眼这个图，我们知道在槽位10的Log，3个副本都有记录，它可能已经commit了，所以我们不能丢弃它。类似的在槽位11的Log，因为它被过半服务器记录了，它也可能commit了，所以我们也不能丢弃它。在槽位12记录的两个Log（分别是任期4和任期5），都没有被commit，所以Raft可以丢弃它们。这里没有要求必须都丢弃它们，但是至少需要丢弃一个Log，因为最终你还是要保持多个副本之间的Log一致。

> 学生提问：槽位10和11的请求必然执行成功了吗？
> Robert教授：对于槽位11，甚至对于槽位10，我们不能从Log中看出来Leader在故障之前到底执行到了哪一步。有一种可能是Leader在发送完AppendEntries之后就立刻故障了，所以Leader没能收到其他副本的确认，相应的请求也就不会commit，进而也就不会执行这个请求，所以它也就不会发出增加了的commit值，其他副本也就可能也没有执行这个请求。所以完全可能槽位10和槽位11的请求没有被执行。如果Raft能知道这些，那么丢弃槽位10和槽位11的Log也是合法的，因为它们没有被commit。但是从Log上看，没有办法否认这些请求被commit了。换句话说，这些请求可能commit了。所以Raft必须认为它们已经被commit了，因为完全有可能，Leader是在对这些请求走完完整流程之后再故障。所以这里，我们不能排除Leader已经返回响应给客户端的可能性，只要这种可能性存在，我们就不能将槽位10和槽位11的Log丢弃，因为客户端可能已经知道了这个请求被执行了。所以我们必须假设这些请求被commit了。

我们会在下一节课继续这个话题。