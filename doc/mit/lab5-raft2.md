## 一 日志恢复

我们现在处于这样一个场景



![img](https://pic3.zhimg.com/v2-3f3b77262ed9704d4315a56862f73e1e_b.jpg)



我们假设下一个任期是6。尽管你无法从黑板上确认这一点，但是下一个任期号至少是6或者更大。我们同时假设S3在任期6被选为Leader。在某个时刻，新Leader S3会发送任期6的第一个AppendEntries RPC，来传输任期6的第一个Log，这个Log应该在槽位13。

这里的AppendEntries消息实际上有两条，因为要发给两个Followers。它们包含了客户端发送给Leader的请求。我们现在想将这个请求复制到所有的Followers上。这里的AppendEntries RPC还包含了prevLogIndex字段和prevLogTerm字段。所以Leader在发送AppendEntries消息时，会附带前一个槽位的信息。在我们的场景中，prevLogIndex是前一个槽位的位置，也就是12；prevLogTerm是S3上前一个槽位的任期号，也就是5。



![img](https://pic3.zhimg.com/v2-38d7975ce732ae668427d723437c4002_b.jpg)



这样的AppendEntries消息发送给了Followers。而Followers，它们在收到AppendEntries消息时，可以知道它们收到了一个带有若干Log条目的消息，并且是从槽位13开始。Followers在写入Log之前，会检查本地的前一个Log条目，是否与Leader发来的有关前一条Log的信息匹配。

所以对于S2 它显然是不匹配的。S2 在槽位12已经有一个条目，但是它来自任期4，而不是任期5。所以S2将拒绝这个AppendEntries，并返回False给Leader。S1在槽位12还没有任何Log，所以S1也将拒绝Leader的这个AppendEntries。到目前位置，一切都还好。为什么这么说呢？因为我们完全不想看到的是，S2 把这条新的Log添加在槽位13。因为这样会破坏Raft论文中图2所依赖的归纳特性，并且隐藏S2 实际上在槽位12有一条不同的Log的这一事实。



![img](https://pic4.zhimg.com/v2-608de615b86e992eaa7657d2d284763f_b.jpg)



所以S1和S2都没有接受这条AppendEntries消息，所以，Leader看到了两个拒绝。

Leader为每个Follower维护了nextIndex。所以它有一个S2的nextIndex，还有一个S1的nextIndex。之前没有说明的是，如果Leader之前发送的是有关槽位13的Log，这意味着Leader对于其他两个服务器的nextIndex都是13。这种情况发生在Leader刚刚当选，因为Raft论文的图2规定了，nextIndex的初始值是从新任Leader的最后一条日志开始，而在我们的场景中，对应的就是槽位13.

为了响应Followers返回的拒绝，Leader会减小对应的nextIndex。所以它现在减小了两个Followers的nextIndex。这一次，Leader发送的AppendEntries消息中，prevLogIndex等于11，prevLogTerm等于3。同时，这次Leader发送的AppendEntries消息包含了prevLogIndex之后的所有条目，也就是S3上槽位12和槽位13的Log。



![img](https://pic4.zhimg.com/v2-13680908f2234401cc2444539010bad7_b.jpg)



对于S2来说，这次收到的AppendEntries消息中，prevLogIndex等于11，prevLogTerm等于3，与自己本地的Log匹配，所以，S2会接受这个消息。Raft论文中的图2规定，如果接受一个AppendEntries消息，那么需要首先删除本地相应的Log（如果有的话），再用AppendEntries中的内容替代本地Log。所以，S2会这么做：它会删除本地槽位12的记录，再添加AppendEntries中的Log条目。这个时候，S2的Log与S3保持了一致。



![img](https://pic3.zhimg.com/v2-5a0d17e256802c59be744a96f27cb422_b.jpg)



但是，S1仍然有问题，因为它的槽位11是空的，所以它不能匹配这次的AppendEntries。它将再次返回False。而Leader会将S1对应的nextIndex变为11，并在AppendEntries消息中带上从槽位11开始之后的Log（也就是槽位11，12，13对应的Log）。并且带上相应的prevLogIndex（10）和prevLogTerm（3）。



![img](https://pic2.zhimg.com/v2-6c607d572987dec21e6746f260d7c1ed_b.jpg)



这次的请求可以被S1接受，并得到肯定的返回。现在它们都有了一致的Log。



![img](https://pic2.zhimg.com/v2-d1a22fdb5e3fa96d17895e4373192779_b.jpg)



而Leader在收到了Followers对于AppendEntries的肯定的返回之后，它会增加相应的nextIndex到14。 



![img](https://pic3.zhimg.com/v2-2fc87ec04174c30ece48912022254b42_b.jpg)



在这里，Leader使用了一种备份机制来探测Followers的Log中，第一个与Leader的Log相同的位置。在获得位置之后，Leader会给Follower发送从这个位置开始的，剩余的全部Log。经过这个过程，所有节点的Log都可以和Leader保持一致。

重复一个我们之前讨论过的话题，或许我们还会再讨论。在刚刚的过程中，我们擦除了一些Log条目，比如我们刚刚删除了S2中的槽位12的Log。这个位置是任期4的Log。现在的问题是，为什么Raft系统可以安全的删除这条记录？毕竟我们在删除这条记录时，某个相关的客户端请求也随之被丢弃了。



![img](https://pic2.zhimg.com/v2-47e3bd1cb2dc6198d0f14f0f8d804e45_b.jpg)



我在上堂课说过这个问题，这里的原理是什么呢？是的，这条Log条目并没有存在于过半服务器中，因此无论之前的Leader是谁，发送了这条Log，它都没有得到过半服务器的认可。因此旧的Leader不可能commit了这条记录，也就不可能将它应用到应用程序的状态中，进而也就不可能回复给客户端说请求成功了。因为它没有存在于过半服务器中，发送这个请求的客户端没有理由认为这个请求被执行了，也不可能得到一个回复。因为这里有一条规则就是，Leader只会在commit之后回复给客户端。客户端甚至都没有理由相信这个请求被任意服务器收到了。并且，Raft论文中的图2说明，如果客户端发送请求之后一段时间没有收到回复，它应该重新发送请求。所以我们知道，不论这个被丢弃的请求是什么，我们都没有执行它，没有把它包含在任何状态中，并且客户端之后会重新发送这个请求。

> 学生提问：前面的过程中，为什么总是删除Followers的Log的结尾部分？
> Robert教授：一个备选的答案是，Leader有完整的Log，所以当Leader收到有关AppendEntries的False返回时，它可以发送完整的日志给Follower。如果你刚刚启动系统，甚至在一开始就发生了非常反常的事情，某个Follower可能会从第一条Log 条目开始恢复，然后让Leader发送整个Log记录，因为Leader有这些记录。如果有必要的话，Leader拥有填充每个节点的日志所需的所有信息。

## 二 快速恢复

在前面（7.1）介绍的日志恢复机制中，如果Log有冲突，Leader每次会回退一条Log条目。 这在许多场景下都没有问题。但是在某些现实的场景中，至少在Lab2的测试用例中，每次回退一条Log条目会花费很长很长的时间。所以，现实的场景中，可能一个Follower关机了很长时间，错过了大量的AppendEntries消息。这时，Leader重启了。按照Raft论文中的图2，如果一个Leader重启了，它会将所有Follower的nextIndex设置为Leader本地Log记录的下一个槽位（7.1有说明）。所以，如果一个Follower关机并错过了1000条Log条目，Leader重启之后，需要每次通过一条RPC来回退一条Log条目来遍历1000条Follower错过的Log记录。这种情况在现实中并非不可能发生。在一些不正常的场景中，假设我们有5个服务器，有1个Leader，这个Leader和另一个Follower困在一个网络分区。但是这个Leader并不知道它已经不再是Leader了。它还是会向它唯一的Follower发送AppendEntries，因为这里没有过半服务器，所以没有一条Log会commit。在另一个有多数服务器的网络分区中，系统选出了新的Leader并继续运行。旧的Leader和它的Follower可能会记录无限多的旧的任期的未commit的Log。当旧的Leader和它的Follower重新加入到集群中时，这些Log需要被删除并覆盖。可能在现实中，这不是那么容易发生，但是你会在Lab2的测试用例中发现这个场景。

所以，为了能够更快的恢复日志，Raft论文在论文的5.3结尾处，对一种方法有一些模糊的描述。原文有些晦涩，在这里我会以一种更好的方式尝试解释论文中有关快速恢复的方法。这里的大致思想是，让Follower返回足够的信息给Leader，这样Leader可以以任期（Term）为单位来回退，而不用每次只回退一条Log条目。所以现在，在恢复Follower的Log时，如果Leader和Follower的Log不匹配，Leader只需要对每个不同的任期发送一条AppendEntries，而不用对每个不同的Log条目发送一条AppendEntries。这只是一种加速策略，当然，或许你也可以想出许多其他不同的日志恢复加速策略。

我将可能出现的场景分成3类，为了简化，这里只画出一个Leader（S2）和一个Follower（S1），S2将要发送一条任期号为6的AppendEntries消息给Follower。

- 场景1：S1没有任期6的任何Log，因此我们需要回退一整个任期的Log。



![img](https://pic3.zhimg.com/v2-808d1e4b13b295f92165060c49519f36_b.jpg)



- 场景2：S1收到了任期4的旧Leader的多条Log，但是作为新Leader，S2只收到了一条任期4的Log。所以这里，我们需要覆盖S1中有关旧Leader的一些Log。



![img](https://pic4.zhimg.com/v2-1c5e4085f5113ac08848a843a3659283_b.jpg)



- 场景3：S1与S2的Log不冲突，但是S1缺失了部分S2中的Log。



![img](https://pic4.zhimg.com/v2-728bf535c581d94600412ae29d251853_b.jpg)



可以让Follower在回复Leader的AppendEntries消息中，携带3个额外的信息，来加速日志的恢复。这里的回复是指，Follower因为Log信息不匹配，拒绝了Leader的AppendEntries之后的回复。这里的三个信息是指：

- XTerm：这个是Follower中与Leader冲突的Log对应的任期号。在之前（7.1）有介绍Leader会在prevLogTerm中带上本地Log记录中，前一条Log的任期号。如果Follower在对应位置的任期号不匹配，它会拒绝Leader的AppendEntries消息，并将自己的任期号放在XTerm中。如果Follower在对应位置没有Log，那么这里会返回 -1。
- XIndex：这个是Follower中，对应任期号为XTerm的第一条Log条目的槽位号。
- XLen：如果Follower在对应位置没有Log，那么XTerm会返回-1，XLen表示空白的Log槽位数。



![img](https://pic3.zhimg.com/v2-03e692c03c43ff907591e8780057393a_b.jpg)



我们再来看这些信息是如何在上面3个场景中，帮助Leader快速回退到适当的Log条目位置。

- 场景1。Follower（S1）会返回XTerm=5，XIndex=2。Leader（S2）发现自己没有任期5的日志，它会将自己本地记录的，S1的nextIndex设置到XIndex，也就是S1中，任期5的第一条Log对应的槽位号。所以，如果Leader完全没有XTerm的任何Log，那么它应该回退到XIndex对应的位置（这样，Leader发出的下一条AppendEntries就可以一次覆盖S1中所有XTerm对应的Log）。
- 场景2。Follower（S1）会返回XTerm=4，XIndex=1。Leader（S2）发现自己其实有任期4的日志，它会将自己本地记录的S1的nextIndex设置到本地在XTerm位置的Log条目后面，也就是槽位2。下一次Leader发出下一条AppendEntries时，就可以一次覆盖S1中槽位2和槽位3对应的Log。
- 场景3。Follower（S1）会返回XTerm=-1，XLen=2。这表示S1中日志太短了，以至于在冲突的位置没有Log条目，Leader应该回退到Follower最后一条Log条目的下一条，也就是槽位2，并从这开始发送AppendEntries消息。槽位2可以从XLen中的数值计算得到。

这些信息在Lab中会有用，如果你错过了我的描述，你可以再看看视频（Robert教授说的）。

对于这里的快速回退机制有什么问题吗？

> 学生提问：这里是线性查找，可以使用类似二分查找的方法进一步加速吗？
> Robert教授：我认为这是对的，或许这里可以用二分查找法。我没有排除其他方法的可能，我的意思是，Raft论文中并没有详细说明是怎么做的，所以我这里加工了一下。或许有更好，更快的方式来完成。如果Follower返回了更多的信息，那是可以用一些更高级的方法，例如二分查找，来完成。
> 为了通过Lab2的测试，你肯定需要做一些优化工作。我们提供的Lab2的测试用例中，有一件不幸但是不可避免的事情是，它们需要一些实时特性。这些测试用例不会永远等待你的代码执行完成并生成结果。所以有可能你的方法技术上是对的，但是花了太多时间导致测试用例退出。这个时候，你是不能通过全部的测试用例的。因此你的确需要关注性能，从而使得你的方案即是正确的，又有足够的性能。不幸的是，性能与Log的复杂度相关，所以很容易就写出一个正确但是不够快的方法出来。
> 学生提问：能在解释一下这里的流程吗？
> Robert教授：这里，Leader发现冲突的方法在于，Follower会返回它从冲突条目中看到的任期号（XTerm）。在场景1中，Follower会设置XTerm=5，因为这是有冲突的Log条目对应的任期号。Leader会发现，哦，我的Log中没有任期5的条目。因此，在场景1中，Leader会一次性回退到Follower在任期5的起始位置。因为Leader并没有任何任期5的Log，所以它要删掉Follower中所有任期5的Log，这通过回退到Follower在任期5的第一条Log条目的位置，也就是XIndex达到的。

## 三 持久化

下一个我想介绍的是持久化存储（persistence）。你可以从Raft论文的图2的左上角看到，有些数据被标记为持久化的（Persistent），有些信息被标记为非持久化的（Volatile）。持久化和非持久化的区别只在服务器重启时重要。当你更改了被标记为持久化的某个数据，服务器应该将更新写入到磁盘，或者其它的持久化存储中，例如一个电池供电的RAM。持久化的存储可以确保当服务器重启时，服务器可以找到相应的数据，并将其加载到内存中。这样可以使得服务器在故障并重启后，继续重启之前的状态。

你或许会认为，如果一个服务器故障了，那简单直接的方法就是将它从集群中摘除。我们需要具备从集群中摘除服务器，替换一个全新的空的服务器，并让该新服务器在集群内工作的能力。实际上，这是至关重要的，因为如果一些服务器遭受了不可恢复的故障，例如磁盘故障，你绝对需要替换这台服务器。同时，如果磁盘故障了，你也不能指望能从该服务器的磁盘中获得任何有用的信息。所以我们的确需要能够用全新的空的服务器替代现有服务器的能力。你或许认为，这就足以应对任何出问题的场景了，但实际上不是的。

实际上，一个常见的故障是断电。断电的时候，整个集群都同时停止运行，这种场景下，我们不能通过从Dell买一些新的服务器来替换现有服务器进而解决问题。这种场景下，如果我们希望我们的服务是容错的， 我们需要能够得到之前状态的拷贝，这样我们才能保持程序继续运行。因此，至少为了处理同时断电的场景，我们不得不让服务器能够将它们的状态存储在某处，这样当供电恢复了之后，还能再次获取这个状态。这里的状态是指，为了让服务器在断电或者整个集群断电后，能够继续运行所必不可少的内容。这是理解持久化存储的一种方式。

在Raft论文的图2中，有且仅有三个数据是需要持久化存储的。它们分别是Log、currentTerm、votedFor。Log是所有的Log条目。当某个服务器刚刚重启，在它加入到Raft集群之前，它必须要检查并确保这些数据有效的存储在它的磁盘上。服务器必须要有某种方式来发现，自己的确有一些持久化存储的状态，而不是一些无意义的数据。



![img](https://gitee.com/zisuu/picture/raw/master/img/20210221203125.jpeg)



Log需要被持久化存储的原因是，这是唯一记录了应用程序状态的地方。Raft论文图2并没有要求我们持久化存储应用程序状态。假如我们运行了一个数据库或者为VMware FT运行了一个Test-and-Set服务，根据Raft论文图2，实际的数据库或者实际的test-set值，并不会被持久化存储，只有Raft的Log被存储了。所以当服务器重启时，唯一能用来重建应用程序状态的信息就是存储在Log中的一系列操作，所以Log必须要被持久化存储。

那currentTerm呢？为什么currentTerm需要被持久化存储？是的，currentTerm和votedFor都是用来确保每个任期只有最多一个Leader。在一个故障的场景中，如果一个服务器收到了一个RequestVote请求，并且为服务器1投票了，之后它故障。如果它没有存储它为哪个服务器投过票，当它故障重启之后，收到了来自服务器2的同一个任期的另一个RequestVote请求，那么它还是会投票给服务器2，因为它发现自己的votedFor是空的，因此它认为自己还没投过票。现在这个服务器，在同一个任期内同时为服务器1和服务器2投了票。因为服务器1和服务器2都会为自己投票，它们都会认为自己有过半选票（3票中的2票），那它们都会成为Leader。现在同一个任期里面有了两个Leader。这就是为什么votedFor必须被持久化存储。

currentTerm的情况要更微妙一些，但是实际上还是为了实现一个任期内最多只有一个Leader，我们之前实际上介绍过这里的内容。如果（重启之后）我们不知道任期号是什么，很难确保一个任期内只有一个Leader。 



![img](https://gitee.com/zisuu/picture/raw/master/img/20210221203125.jpeg)



在这里例子中，S1关机了，S2和S3会尝试选举一个新的Leader。它们需要证据证明，正确的任期号是8，而不是6。如果仅仅是S2和S3为彼此投票，它们不知道当前的任期号，它们只能查看自己的Log，它们或许会认为下一个任期是6（因为Log里的上一个任期是5）。如果它们这么做了，那么它们会从任期6开始添加Log。但是接下来，就会有问题了，因为我们有了两个不同的任期6（另一个在S1中）。这就是为什么currentTerm需要被持久化存储的原因，因为它需要用来保存已经被使用过的任期号。

这些数据需要在每次你修改它们的时候存储起来。所以可以确定的是，安全的做法是每次你添加一个Log条目，更新currentTerm或者更新votedFor，你或许都需要持久化存储这些数据。在一个真实的Raft服务器上，这意味着将数据写入磁盘，所以你需要一些文件来记录这些数据。如果你发现，直到服务器与外界通信时，才有可能持久化存储数据，那么你可以通过一些批量操作来提升性能。例如，只在服务器回复一个RPC或者发送一个RPC时，服务器才进行持久化存储，这样可以节省一些持久化存储的操作。

之所以这很重要是因为，向磁盘写数据是一个代价很高的操作。如果是一个机械硬盘，我们通过写文件的方式来持久化存储，向磁盘写入任何数据都需要花费大概10毫秒时间。因为你要么需要等磁盘将你想写入的位置转到磁针下面， 而磁盘大概每10毫秒转一次。要么，就是另一种情况更糟糕，磁盘需要将磁针移到正确的轨道上。所以这里的持久化操作的代价可能会非常非常高。对于一些简单的设计，这些操作可能成为限制性能的因素，因为它们意味着在这些Raft服务器上执行任何操作，都需要10毫秒。而10毫秒相比发送RPC或者其他操作来说都太长了。如果你持久化存储在一个机械硬盘上，那么每个操作至少要10毫秒，这意味着你永远也不可能构建一个每秒能处理超过100个请求的Raft服务。这就是所谓的synchronous disk updates的代价。它存在于很多系统中，例如运行在你的笔记本上的文件系统。



![img](https://gitee.com/zisuu/picture/raw/master/img/20210221203125.jpeg)



设计人员花费了大量的时间来避开synchronous disk updates带来的性能问题。为了让磁盘的数据保证安全，同时为了能安全更新你的笔记本上的磁盘，文件系统对于写入操作十分小心，有时需要等待磁盘（前一个）写入完成。所以这（优化磁盘写入性能）是一个出现在所有系统中的常见的问题，也必然出现在Raft中。

如果你想构建一个能每秒处理超过100个请求的系统，这里有多个选择。其中一个就是，你可以使用SSD硬盘，或者某种闪存。SSD可以在0.1毫秒完成对于闪存的一次写操作，所以这里性能就提高了100倍。更高级一点的方法是，你可以构建一个电池供电的DRAM，然后在这个电池供电的DRAM中做持久化存储。这样，如果Server重启了，并且重启时间短于电池的可供电时间，这样你存储在RAM中的数据还能保存。如果资金充足，且不怕复杂的话，这种方式的优点是，你可以每秒写DRAM数百万次，那么持久化存储就不再会是一个性能瓶颈。所以，synchronous disk updates是为什么数据要区分持久化和非持久化（而非所有的都做持久化）的原因（越少数据持久化，越高的性能）。Raft论文图2考虑了很多性能，故障恢复，正确性的问题。

有任何有关持久化存储的问题吗？

> 学生提问：当你写你的Raft代码时，你实际上需要确认，当你持久化存储一个Log或者currentTerm，这些数据是否实时的存储在磁盘中，你该怎么做来确保它们在那呢？
> Robert教授：在一个UNIX或者一个Linux或者一个Mac上，为了调用系统写磁盘的操作，你只需要调用write函数，在write函数返回时，并不能确保数据存在磁盘上，并且在重启之后还存在。几乎可以确定（write返回之后）数据不会在磁盘上。所以，如果在UNIX上，你调用了write，将一些数据写入之后，你需要调用fsync。在大部分系统上，fsync可以确保在返回时，所有之前写入的数据已经安全的存储在磁盘的介质上了。之后，如果机器重启了，这些信息还能在磁盘上找到。fsync是一个代价很高的调用，这就是为什么它是一个独立的函数，也是为什么write不负责将数据写入磁盘，fsync负责将数据写入磁盘。因为写入磁盘的代价很高，你永远也不会想要执行这个操作，除非你想要持久化存储一些数据。



![img](https://gitee.com/zisuu/picture/raw/master/img/20210221203125.jpeg)



所以你可以使用一些更贵的磁盘。另一个常见方法是，批量执行操作。如果有大量的客户端请求，或许你应该同时接收它们，但是先不返回。等大量的请求累积之后，一次性持久化存储（比如）100个Log，之后再发送AppendEntries。如果Leader收到了一个客户端请求，在发送AppendEntries RPC给Followers之前，必须要先持久化存储在本地。因为Leader必须要commit那个请求，并且不能忘记这个请求。实际上，在回复AppendEntries 消息之前，Followers也需要持久化存储这些Log条目到本地，因为它们最终也要commit这个请求，它们不能因为重启而忘记这个请求。

最后，有关持久化存储，还有一些细节。有些数据在Raft论文的图2中标记为非持久化的。所以，这里值得思考一下，为什么服务器重启时，commitIndex、lastApplied、nextIndex、matchIndex，可以被丢弃？例如，lastApplied表示当前服务器执行到哪一步，如果我们丢弃了它的话，我们需要重复执行Log条目两次（重启前执行过一次，重启后又要再执行一次），这是正确的吗？为什么可以安全的丢弃lastApplied？

这里综合考虑了Raft的简单性和安全性。之所以这些数据是非持久化存储的，是因为Leader可以通过检查自己的Log和发送给Followers的AppendEntries的结果，来发现哪些内容已经commit了。如果因为断电，所有节点都重启了。Leader并不知道哪些内容被commit了，哪些内容被执行了。但是当它发出AppendEntries，并从Followers搜集回信息。它会发现，Followers中有哪些Log与Leader的Log匹配，因此也就可以发现，在重启前，有哪些被commit了。

另外，Raft论文的图2假设，应用程序状态会随着重启而消失。所以图2认为，既然Log已经持久化存储了，那么应用程序状态就不必再持久化存储。因为在图2中，Log从系统运行的初始就被持久化存储下来。所以，当Leader重启时，Leader会从第一条Log开始，执行每一条Log条目，并提交给应用程序。所以，重启之后，应用程序可以通过重复执行每一条Log来完全从头构建自己的状态。这是一种简单且优雅的方法，但是很明显会很慢。这将会引出我们的下一个话题：Log compaction和Snapshot。