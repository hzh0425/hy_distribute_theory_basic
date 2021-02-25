## Jraft日志复制过程图解

SOFAJRaft 是对 Raft 共识算法的 Java 实现。既然是共识算法，就不可避免的要对需要达成共识的内容在多个服务器节点之间进行传输，在 SOFAJRaft 中我们将这些内容封装成一个个日志块 (LogEntry)，这种服务器节点间的日志传输行为在 SOFAJRaft 中也就有了专门的术语：**日志复制**。

### 棋盘问题

为了便于阅读理解，我们用一个象棋的故事来类比日志复制的流程和可能遇到的问题。

假设我们穿越到古代，要为一场即将举办的象棋比赛设计直播方案。当然所有电子通讯技术此时都已经不可用了，幸好象棋比赛是一种能用精简的文字描述赛况的项目，比如：“炮二平五”, “马８进７”, “车２退３”等，我们将这些描述性文字称为**棋谱**。这样只要我们在场外同样摆上棋盘 (可能很大，方便围观)，通过棋谱就可以把棋手的对弈过程直播出来。

![图片](https://mmbiz.qpic.cn/mmbiz_png/nibOZpaQKw0icSopE8dboCY4pWXeusPVwaGlze72PjWGG0FCEx5HEdGoIDAHzjAIvs206MeqdqwdTY2PTzohhzWQ/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

图1 - 通过棋谱直播

所以我们的直播方案就是：赛场内两位棋手正常对弈，设一个专门的记录员来记录棋手走出的每一步，安排一个旗童飞奔于赛场内外，棋手每走一步，旗童就将其以棋谱的方式传递给场外，这样观众就能在场外准实时的观看对弈的过程，获得同观看直播相同的体验。

![img](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)

图2 - 一个简单的直播方案

这便是 SOFAJRaft 日志复制的人肉版，接下来我们完善一下这个“直播系统”，让它逐步对齐真实的日志复制。



#### **改进1. 增加记录员的数量** 



假设我们的比赛获得了很高的关注度，我们需要在赛场外摆出更多的直播场地以供更多的观众观看。

![图片](https://mmbiz.qpic.cn/mmbiz_png/nibOZpaQKw0icSopE8dboCY4pWXeusPVwaRx5icjJB6cWolLib72K14JZ8DxmoIE1DXa4IkP0tpOWGXZOA0t1usxbw/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

图3 - 更多的直播平台

这样我们就要安排更多的旗童来传递棋谱，场外的每一台直播都需要一个旗童来负责，这些旗童不停的在赛场内外奔跑传递棋谱信息。有的直播平台离赛场远一些，旗童要跑很久才行，相应的直播延迟就会大一些，而有些直播平台离得很近，对应的旗童就能很快的将对弈情况同步到直播。

随着直播场地的增加，负责记录棋局的记录员的压力就会增加，因为他要针对不同的旗童每次提供不同的棋谱内容，有的慢有的快。如果记录员一旦记混了或者眼花了，就会出现严重的直播事故(观众看到的不再是棋手真正的棋局)。

![图片](https://mmbiz.qpic.cn/mmbiz_png/nibOZpaQKw0icSopE8dboCY4pWXeusPVwaHzotS5qoF0MAsKbJEUCeI8GYf1u8KfSicSxtQ9icKxlwHR0HDILwusHg/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

图4 - 压力很大的记录员

为此我们要作出一些优化，为每个场外的直播平台安排一个专门的记录员，这样 “赛局-记录员-旗童-直播局” 就构成了单线模式，专人专职高效可靠。

![图片](https://mmbiz.qpic.cn/mmbiz_png/nibOZpaQKw0icSopE8dboCY4pWXeusPVwawvjQVln2YndMcKCHM7BJCO4EtVXO9LNOicnQ2cGWoJ9fZnRtkr3WERA/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

图5 - “赛局-记录员-旗童-直播棋局”



#### **改进2. 增加旗童每次传递的信息量** 



起初我们要求棋手每走一步，旗童就向外传递一次棋谱。可是随着比赛进行，其弊端也逐渐显现，一方面记录员记录了很多棋局信息没有传递出去，以至于不得不请求棋手停下来等待 (不可思议)；另一方面，场外的观众对于这种“卡帧”的直播模式也很不满意。



所以我们做出改进，要求旗童每次多记几步棋，这样记录员不会积攒太多的待直播信息，观众也能一次看到好几步，而这对于聪明的旗童来说并不是什么难事，如此改进达到了共赢的局面。

![图片](https://mmbiz.qpic.cn/mmbiz_png/nibOZpaQKw0icSopE8dboCY4pWXeusPVwaMic8NOIBrvs3eK4oeH8aicqIWXMKywa4ezbbl5SdJnCseogMwluvWeibg/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

图6 - 旗童批量携带信息



#### **改进3. 增加快照模式** 



棋局愈发精彩，应棋迷的强烈要求，我们临时增加了几个直播场地，这时棋手已经走了很多步了，按照我们的常规手段，负责新直播的记录员和旗童需要把过去的每一步都在直播棋盘上还原一遍(回放的过程)，与此同时棋手还在不断下出新的内容。

从直觉上来说这也是一种很不聪明的方式，所以这时我们采用快照模式，不再要求旗童传递过去的每一步棋谱，而是把当前的棋局图直接描下来，旗童将图带出去后，按照图谱直接摆子。这样新直播平台就能快速追上棋局进度，让观众欣赏到赛场同步的棋局对弈了。

![图片](https://mmbiz.qpic.cn/mmbiz_png/nibOZpaQKw0icSopE8dboCY4pWXeusPVwah6n6Rm8VicUWVcSkwmfmnJ6yksgrkscb2OY2d0wkJhvsRE2rwuSRJvw/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

图7 - 采用快照模式



**改进4. 每一个直播平台用多个旗童传递信息**



虽然我们之前已经在改进 2 中增加了旗童每次携带的信息量，但是在一些情况下(棋手下快棋、直播平台很远等)，记录员依然无法将信息及时同步给场外。这时我们需要增加多个旗童，各旗童有次序的将信息携带到场外，这样记录员就可以更快速的把信息同步给场外直播平台。

![图片](https://mmbiz.qpic.cn/mmbiz_png/nibOZpaQKw0icSopE8dboCY4pWXeusPVwar5trbeohGRl9b5wMBvsNneaibV0aEiactG4G2IicN7PPf3pMIicOoPnDrw/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

图8 - 利用多个旗童传递信息，实现 pipeline 效果

现在这个人肉的直播平台在我们的逐步改进下已经具备了 SOFAJRaft 日志复制的下面几个主要特点：



#### **特点1：被复制的日志是有序且连续的** 



如果棋谱传递的顺序不一样，最后下出的棋局可能也是完全不同的。而 SOFAJRaft 在日志复制时，其日志传输的顺序也要保证严格的顺序，所有日志既不能乱序也不能有空洞 (也就是说不能被漏掉)。

![图片](https://mmbiz.qpic.cn/mmbiz_png/nibOZpaQKw0icSopE8dboCY4pWXeusPVwa7zwQ3q3THXK1aPrhsQUjNrCROcQkUxBP3twUHAq5bb79QA3enZl4gw/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

图9 - 日志保持严格有序且连续



#### **特点2：复制日志是并发的** 



SOFAJRaft 中 Leader 节点会同时向多个 Follower 节点复制日志，在 Leader 中为每一个 Follower 分配一个 Replicator，专用来处理复制日志任务。在棋局中我们也针对每个直播平台安排一个记录员，用来将对弈棋谱同步给对应的直播平台。

![图片](https://mmbiz.qpic.cn/mmbiz_png/nibOZpaQKw0icSopE8dboCY4pWXeusPVwaYs31P81HksATNOywm6duVzN3TxlX6oosjbUDcBcjQsC0fuvicceFqug/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

图10 - 并发复制日志



#### **特点3：复制日志是批量的** 



SOFAJRaft 中 Leader 节点会将日志成批的复制给 Follower，就像旗童会每次携带多步棋信息到场外。

![img](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)

图11 - 日志被批量复制



#### **特点4：日志复制中的快照** 



在改进 3 中，我们让新加入的直播平台直接复制当前的棋局，而不再回放过去的每一步棋谱，这就是 SOFAJRaft 中的快照 (Snapshot) 机制。用 Snapshot 能够让 Follower 快速跟上 Leader 的日志进度，不再回放很早以前的日志信息，即缓解了网络的吞吐量，又提升了日志同步的效率。



#### **特点5：复制日志的 pipeline 机制** 



在改进 4 中，我们让多个旗童参与信息传递，这样记录员和直播平台间就可以以“流式”的方式传递信息，这样既能保证信息传递有序也能保证信息传递持续。

在 SOFAJRaft 中我们也有类似的机制来保证日志复制流式的进行，这种机制就是 pipeline。Pipeline 使得 Leader 和 Follower 双方不再需要严格遵从 “Request - Response - Request” 的交互模式，Leader 可以在没有收到 Response 的情况下，持续的将复制日志的 AppendEntriesRequest 发送给 Follower。

在具体实现时，Leader 只需要针对每个 Follower 维护一个队列，记录下已经复制的日志，如果有日志复制失败的情况，就将其后的日志重发给 Follower。这样就能保证日志复制的可靠性，具体细节我们在源码解析中再谈。

![图片](https://mmbiz.qpic.cn/mmbiz_png/nibOZpaQKw0icSopE8dboCY4pWXeusPVwaTVzo7AO1qHo5vPVzZH2Ak9nA5ZuH9sV6a93l905NicktZsZSAQHZ9Pw/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

图12 - 日志复制的 pipeline 机制



### **源码解析** 



上面就是日志复制在原理层面的介绍，而在代码实现中主要是由 `Replicator` 和 `NodeImpl` 来分别实现 Leader 和 Follower 的各自逻辑，主要的方法列于下方。在处理源码中有三点值得我们关注。

![图片](https://mmbiz.qpic.cn/mmbiz_png/nibOZpaQKw0icSopE8dboCY4pWXeusPVwaT1xlBkg49XYrhsmK39TgC6ZoEibKCpO8LLMibdjCiavhfpq5XEFToxLQA/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

图13 - 相关的方法

#### **关注1: Replicator 的 Probe 状态**

![图片](https://mmbiz.qpic.cn/mmbiz_png/nibOZpaQKw0icSopE8dboCY4pWXeusPVwac4SIQSl9yicQictYib8bGoCdYH6S5MWJHicIJCZN8NQ7hyA9whwtcOAZVQ/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

图14 - Replicator 的状态

Leader 节点在通过 Replicator 和 Follower 建立连接之后，要发送一个 Probe 类型的探针请求，目的是知道 Follower 已经拥有的的日志位置，以便于向 Follower 发送后续的日志。

![图片](https://mmbiz.qpic.cn/mmbiz_png/nibOZpaQKw0icSopE8dboCY4pWXeusPVwaky0hl75fF8bavkX5MwfQKrgLqKPUhH2yPF5xH4e6SAgwvuiabElVejw/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

图15 - 发送探针来知道 follower 的 logindex

#### 关注2: 用 Inflight 来辅助实现 pipeline

Inflight 是对批量发送出去的 logEntry 的一种抽象，他表示哪些 logEntry 已经被封装成日志复制 request 发送出去了。

![图片](https://mmbiz.qpic.cn/mmbiz_png/nibOZpaQKw0icSopE8dboCY4pWXeusPVwaeYjP7fnwicztic56ZntEDCIp4K94WyJXSdCUpTFibETsPZeV2k1uzI8sQ/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

图16 - Inflight 结构

Leader 维护一个 queue，每发出一批 logEntry 就向 queue 中 添加一个代表这一批 logEntry 的 Inflight，这样当它知道某一批 logEntry 复制失败之后，就可以依赖 queue 中的 Inflight 把该批次 logEntry 以及后续的所有日志重新复制给 follower。既保证日志复制能够完成，又保证了复制日志的顺序不变。

这部分从逻辑上来说比较清晰，但是代码层面需要考虑的东西比较多，所以我们在此处贴出源码，读者可以在源码中继续探索。

![图片](https://mmbiz.qpic.cn/mmbiz_png/nibOZpaQKw0icSopE8dboCY4pWXeusPVwaYsXncZKJcPibBR0OZMEP0pbrX1smnjsIvCnUkWGFsoWDiblk4WuqibRLQ/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

图17 - 复制日志的主要方法

![图片](https://mmbiz.qpic.cn/mmbiz_png/nibOZpaQKw0icSopE8dboCY4pWXeusPVwaRMF7EklMVgLmIcMoXC7YHXHBlw2jcSJ83jKqTow3UDaXpzHd74rx5w/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

图18 - 添加 Inflight 到队列中

当然在日志复制中其实还要考虑更加复杂的情况，比如一旦发生切换 leader 的情况，follower 该如何应对，这些问题希望大家能够进入源码来寻找答案。

#### 关注3: 通信层采用单线程 & 单链接

在 pipeline 机制中，虽然我们在 SOFAJRaft 层面通过 Inflight 队列保证了日志是被有序的复制，对于乱序传输的 LogEntry 通过各种异常流程去排除掉，但是这些被排除掉的乱序日志最终还是要通过重传来保证最终成功，这就会影响日志复制的效率。

![图片](https://mmbiz.qpic.cn/mmbiz_png/nibOZpaQKw0icSopE8dboCY4pWXeusPVwajCwLziabagWbJsd6YBmCIwibHib8uiaiaJPg13A5gfTzludPrl9gs8PRLoQ/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

图19 - 通信层不能保证有序

如上图所示，发送端的 Connection Pool 和 接收端的 Thread Pool 都会让原本“单行道”上有序传输的日志进入“多车道”，因而无法保证有序。所以在通信层面 SOFAJRaft 做了两部分优化去尽量保证 LogEntry 在传输中不会乱序。

\1. 在 Replicator 端，通过 uniqueKey 对日志传输所用的 Url 进行特殊标识 ，这样 SOFABolt (SOFAJRaft 底层所采用的通信框架) 就会为这种 Url 建立单一的连接，也就是发送端的 Connection Pool 中只有一条可用连接。

![图片](https://mmbiz.qpic.cn/mmbiz_png/nibOZpaQKw0icSopE8dboCY4pWXeusPVwa8ibFg3MDWCHj8Y8Y3ZhloqE59jeOXI5Is5O1EuNymLCtjf0QWeiazqYw/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

图20 - 通过 uniqueKey 定制 Url

\2. 在接收端不采用线程池派发任务，增加判断 _dispatch_msg_list_in_default_executor_ 使得我们可以通过 io 线程直接将任务投递到 Processor 中。我们对 SOFABolt 做过一些功能增强，这里提供相关 PR #84 ，有兴趣的读者可以前往了解。

PR #84：

https://github.com/sofastack/sofa-bolt/pull/84

![图片](https://mmbiz.qpic.cn/mmbiz_png/nibOZpaQKw0icSopE8dboCY4pWXeusPVwahHl6Zicxx9bOypbMUDibvDQ4vHQwsf3rXIZicS7z8aLHsEHOs3EVGPnkA/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

图21 - SOFABolt 利用 IO 线程派发 AppendEntriesRequest 到 Processor

这样日志复制的通信模型就变成了我们期望的“单行道”的模式。这种“单行道”能够很大程度上保证传输的日志是有序且连续的，从而提升了 pipeline 的效率。

![图片](https://mmbiz.qpic.cn/mmbiz_png/nibOZpaQKw0icSopE8dboCY4pWXeusPVwa8MvicbFicje4vNOVaob2B2xSU4n8wIHLBPW7wzT0Wtv4Dg7y90y31gCQ/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

# 流程分析

## Leader发送探针获取Follower的LastLogIndex[#](https://www.cnblogs.com/luozhiyun/p/12005975.html#526212271)

Leader 节点在通过 Replicator 和 Follower 建立连接之后，要发送一个 Probe 类型的探针请求，目的是知道 Follower 已经拥有的的日志位置，以便于向 Follower 发送后续的日志。

大致的流程如下：

```
CopyNodeImpl#becomeLeader->replicatorGroup#addReplicator->Replicator#start->Replicator#sendEmptyEntries
```

最后会通过调用Replicator的sendEmptyEntries方法来发送探针来获取Follower的LastLogIndex

**Replicator#sendEmptyEntries**

```java
Copyprivate void sendEmptyEntries(final boolean isHeartbeat,
                              final RpcResponseClosure<AppendEntriesResponse> heartBeatClosure) {
    final AppendEntriesRequest.Builder rb = AppendEntriesRequest.newBuilder();
    //将集群配置设置到rb中，例如Term，GroupId，ServerId等
    if (!fillCommonFields(rb, this.nextIndex - 1, isHeartbeat)) {
        // id is unlock in installSnapshot
        installSnapshot();
        if (isHeartbeat && heartBeatClosure != null) {
            Utils.runClosureInThread(heartBeatClosure, new Status(RaftError.EAGAIN,
                "Fail to send heartbeat to peer %s", this.options.getPeerId()));
        }
        return;
    }
    try {
        final long monotonicSendTimeMs = Utils.monotonicMs();
        final AppendEntriesRequest request = rb.build();

        if (isHeartbeat) {
            ....//省略心跳代码
        } else {
            //statInfo这个类没看到哪里有用到，
            // Sending a probe request.
            //leader发送探针获取Follower的LastLogIndex
            this.statInfo.runningState = RunningState.APPENDING_ENTRIES;
            //将lastLogIndex设置为比firstLogIndex小1
            this.statInfo.firstLogIndex = this.nextIndex;
            this.statInfo.lastLogIndex = this.nextIndex - 1;
            this.appendEntriesCounter++;
            //设置当前Replicator为发送探针
            this.state = State.Probe;
            final int stateVersion = this.version;
            //返回reqSeq，并将reqSeq加一
            final int seq = getAndIncrementReqSeq();
            final Future<Message> rpcFuture = this.rpcService.appendEntries(this.options.getPeerId().getEndpoint(),
                request, -1, new RpcResponseClosureAdapter<AppendEntriesResponse>() {

                    @Override
                    public void run(final Status status) {
                        onRpcReturned(Replicator.this.id, RequestType.AppendEntries, status, request,
                            getResponse(), seq, stateVersion, monotonicSendTimeMs);
                    }

                });
            //Inflight 是对批量发送出去的 logEntry 的一种抽象，他表示哪些 logEntry 已经被封装成日志复制 request 发送出去了
            //这里是将logEntry封装到Inflight中
            addInflight(RequestType.AppendEntries, this.nextIndex, 0, 0, seq, rpcFuture);
        }
        LOG.debug("Node {} send HeartbeatRequest to {} term {} lastCommittedIndex {}", this.options.getNode()
            .getNodeId(), this.options.getPeerId(), this.options.getTerm(), request.getCommittedIndex());
    } finally {
        this.id.unlock();
    }
}
```

在调用sendEmptyEntries方法的时候，会传入isHeartbeat为false和heartBeatClosure为null，因为我们这个方法主要是发送探针获取Follower的位移。
首先调用fillCommonFields方法，将任期，groupId，ServerId，PeerIdLogIndex等设置到rb中，如：

```java
Copyprivate boolean fillCommonFields(final AppendEntriesRequest.Builder rb, long prevLogIndex, final boolean isHeartbeat) {
    final long prevLogTerm = this.options.getLogManager().getTerm(prevLogIndex);
    ....
    rb.setTerm(this.options.getTerm());
    rb.setGroupId(this.options.getGroupId());
    rb.setServerId(this.options.getServerId().toString());
    rb.setPeerId(this.options.getPeerId().toString());
    rb.setPrevLogIndex(prevLogIndex);
    rb.setPrevLogTerm(prevLogTerm);
    rb.setCommittedIndex(this.options.getBallotBox().getLastCommittedIndex());
    return true;
}
```

注意prevLogIndex是nextIndex-1，表示当前的index
继续往下走，会设置statInfo实例里面的属性，但是statInfo这个对象我没看到哪里有用到过。
然后向该Follower发送一个AppendEntriesRequest请求，onRpcReturned负责响应请求。
发送完请求后调用addInflight初始化一个Inflight实例，加入到inflights集合中，如下：

```java
Copyprivate void addInflight(final RequestType reqType, final long startIndex, final int count, final int size,
                         final int seq, final Future<Message> rpcInfly) {
    this.rpcInFly = new Inflight(reqType, startIndex, count, size, seq, rpcInfly);
    this.inflights.add(this.rpcInFly);
    this.nodeMetrics.recordSize("replicate-inflights-count", this.inflights.size());
}
```

Inflight 是对批量发送出去的 logEntry 的一种抽象，他表示哪些 logEntry 已经被封装成日志复制 request 发送出去了，这里是将logEntry封装到Inflight中。

## Leader批量的发送日志给Follower[#](https://www.cnblogs.com/luozhiyun/p/12005975.html#1951673912)

**Replicator#sendEntries**

```java
Copyprivate boolean sendEntries(final long nextSendingIndex) {
    final AppendEntriesRequest.Builder rb = AppendEntriesRequest.newBuilder();
    //填写当前Replicator的配置信息到rb中
    if (!fillCommonFields(rb, nextSendingIndex - 1, false)) {
        // unlock id in installSnapshot
        installSnapshot();
        return false;
    }

    ByteBufferCollector dataBuf = null;
    //获取最大的size为1024
    final int maxEntriesSize = this.raftOptions.getMaxEntriesSize();

    //这里使用了类似对象池的技术，避免重复创建对象
    final RecyclableByteBufferList byteBufList = RecyclableByteBufferList.newInstance();
    try {
        //循环遍历出所有的logEntry封装到byteBufList和emb中
        for (int i = 0; i < maxEntriesSize; i++) {
            final RaftOutter.EntryMeta.Builder emb = RaftOutter.EntryMeta.newBuilder();
            //nextSendingIndex代表下一个要发送的index，i代表偏移量
            if (!prepareEntry(nextSendingIndex, i, emb, byteBufList)) {
                break;
            }
            rb.addEntries(emb.build());
        }
        //如果EntriesCount为0的话，说明LogManager里暂时没有新数据
        if (rb.getEntriesCount() == 0) {
            if (nextSendingIndex < this.options.getLogManager().getFirstLogIndex()) {
                installSnapshot();
                return false;
            }
            // _id is unlock in _wait_more
            waitMoreEntries(nextSendingIndex);
            return false;
        }
        //将byteBufList里面的数据放入到rb中
        if (byteBufList.getCapacity() > 0) {
            dataBuf = ByteBufferCollector.allocateByRecyclers(byteBufList.getCapacity());
            for (final ByteBuffer b : byteBufList) {
                dataBuf.put(b);
            }
            final ByteBuffer buf = dataBuf.getBuffer();
            buf.flip();
            rb.setData(ZeroByteStringHelper.wrap(buf));
        }
    } finally {
        //回收一下byteBufList
        RecycleUtil.recycle(byteBufList);
    }

    final AppendEntriesRequest request = rb.build();
    if (LOG.isDebugEnabled()) {
        LOG.debug(
            "Node {} send AppendEntriesRequest to {} term {} lastCommittedIndex {} prevLogIndex {} prevLogTerm {} logIndex {} count {}",
            this.options.getNode().getNodeId(), this.options.getPeerId(), this.options.getTerm(),
            request.getCommittedIndex(), request.getPrevLogIndex(), request.getPrevLogTerm(), nextSendingIndex,
            request.getEntriesCount());
    }
    //statInfo没找到哪里有用到过
    this.statInfo.runningState = RunningState.APPENDING_ENTRIES;
    this.statInfo.firstLogIndex = rb.getPrevLogIndex() + 1;
    this.statInfo.lastLogIndex = rb.getPrevLogIndex() + rb.getEntriesCount();

    final Recyclable recyclable = dataBuf;
    final int v = this.version;
    final long monotonicSendTimeMs = Utils.monotonicMs();
    final int seq = getAndIncrementReqSeq();
    final Future<Message> rpcFuture = this.rpcService.appendEntries(this.options.getPeerId().getEndpoint(),
        request, -1, new RpcResponseClosureAdapter<AppendEntriesResponse>() {

            @Override
            public void run(final Status status) {
                //回收资源
                RecycleUtil.recycle(recyclable);
                onRpcReturned(Replicator.this.id, RequestType.AppendEntries, status, request, getResponse(), seq,
                    v, monotonicSendTimeMs);
            }

        });
    //添加Inflight
    addInflight(RequestType.AppendEntries, nextSendingIndex, request.getEntriesCount(), request.getData().size(),
        seq, rpcFuture);
    return true;

}
```

1. 首先会调用fillCommonFields方法，填写当前Replicator的配置信息到rb中；
2. 调用prepareEntry，根据当前的I和nextSendingIndex计算出当前的偏移量，然后去LogManager找到对应的LogEntry，再把LogEntry里面的属性设置到emb中，并把LogEntry里面的数据加入到RecyclableByteBufferList中；
3. 如果LogEntry里面没有新的数据，那么EntriesCount会为0，那么就返回；
4. 遍历byteBufList里面的数据，将数据添加到rb中，这样rb里面的数据就是前面是任期、类型、数据长度等信息，rb后面就是真正的数据；
5. 新建AppendEntriesRequest实例发送请求；
6. 添加 Inflight 到队列中。Leader 维护一个 queue，每发出一批 logEntry 就向 queue 中 添加一个代表这一批 logEntry 的 Inflight，这样当它知道某一批 logEntry 复制失败之后，就可以依赖 queue 中的 Inflight 把该批次 logEntry 以及后续的所有日志重新复制给 follower。既保证日志复制能够完成，又保证了复制日志的顺序不变

其中RecyclableByteBufferList采用对象池进行实例化，对象池的相关信息可以看我这篇：[7. SOFAJRaft源码分析—如何实现一个轻量级的对象池？](https://www.cnblogs.com/luozhiyun/p/11924850.html)

下面我们详解一下sendEntries里面的具体方法。

### prepareEntry填充emb属性[#](https://www.cnblogs.com/luozhiyun/p/12005975.html#3558940739)

**Replicator#prepareEntry**

```java
Copyboolean prepareEntry(final long nextSendingIndex, final int offset, final RaftOutter.EntryMeta.Builder emb,
                     final RecyclableByteBufferList dateBuffer) {
    if (dateBuffer.getCapacity() >= this.raftOptions.getMaxBodySize()) {
        return false;
    }
    //设置当前要发送的index
    final long logIndex = nextSendingIndex + offset;
    //如果这个index已经在LogManager中找不到了，那么直接返回
    final LogEntry entry = this.options.getLogManager().getEntry(logIndex);
    if (entry == null) {
        return false;
    }
    //下面就是把LogEntry里面的属性设置到emb中
    emb.setTerm(entry.getId().getTerm());
    if (entry.hasChecksum()) {
        emb.setChecksum(entry.getChecksum()); //since 1.2.6
    }
    emb.setType(entry.getType());
    if (entry.getPeers() != null) {
        Requires.requireTrue(!entry.getPeers().isEmpty(), "Empty peers at logIndex=%d", logIndex);
        for (final PeerId peer : entry.getPeers()) {
            emb.addPeers(peer.toString());
        }
        if (entry.getOldPeers() != null) {
            for (final PeerId peer : entry.getOldPeers()) {
                emb.addOldPeers(peer.toString());
            }
        }
    } else {
        Requires.requireTrue(entry.getType() != EnumOutter.EntryType.ENTRY_TYPE_CONFIGURATION,
            "Empty peers but is ENTRY_TYPE_CONFIGURATION type at logIndex=%d", logIndex);
    }
    final int remaining = entry.getData() != null ? entry.getData().remaining() : 0;
    emb.setDataLen(remaining);
    //把LogEntry里面的数据放入到dateBuffer中
    if (entry.getData() != null) {
        // should slice entry data
        dateBuffer.add(entry.getData().slice());
    }
    return true;
}
```

1. 对比一下传入的dateBuffer的容量是否已经超过了系统设置的容量（512 * 1024），如果超过了则返回false
2. 根据给定的起始的index和偏移量offset计算logIndex，然后去LogManager里面根据index获取LogEntry，如果返回的为则说明找不到了，那么就直接返回false，外层的if判断会执行break跳出循环
3. 然后将LogEntry里面的属性设置到emb对象中，最后将LogEntry里面的数据添加到dateBuffer，这里要做到数据和属性分离

## Follower处理Leader发送的日志复制请求[#](https://www.cnblogs.com/luozhiyun/p/12005975.html#4133977699)

在leader发送完AppendEntriesRequest请求之后，请求的数据会在Follower中被AppendEntriesRequestProcessor所处理

具体的处理方法是processRequest0

```java
Copypublic Message processRequest0(final RaftServerService service, final AppendEntriesRequest request,
                               final RpcRequestClosure done) {

    final Node node = (Node) service;

    //默认使用pipeline
    if (node.getRaftOptions().isReplicatorPipeline()) {
        final String groupId = request.getGroupId();
        final String peerId = request.getPeerId();
        //获取请求的次数，以groupId+peerId为一个维度
        final int reqSequence = getAndIncrementSequence(groupId, peerId, done.getBizContext().getConnection());
        //Follower处理leader发过来的日志请求
        final Message response = service.handleAppendEntriesRequest(request, new SequenceRpcRequestClosure(done,
            reqSequence, groupId, peerId));
        //正常的数据只返回null，异常的数据会返回response
        if (response != null) {
            sendSequenceResponse(groupId, peerId, reqSequence, done.getAsyncContext(), done.getBizContext(),
                response);
        }
        return null;
    } else {
        return service.handleAppendEntriesRequest(request, done);
    }
}
```

调用service的handleAppendEntriesRequest会调用到NodeIml的handleAppendEntriesRequest方法中，handleAppendEntriesRequest方法只是异常情况和leader没有发送数据时才会返回，正常情况是返回null

### 处理响应日志复制请求[#](https://www.cnblogs.com/luozhiyun/p/12005975.html#1721964092)

**NodeIml#handleAppendEntriesRequest**

```java
Copypublic Message handleAppendEntriesRequest(final AppendEntriesRequest request, final RpcRequestClosure done) {
    boolean doUnlock = true;
    final long startMs = Utils.monotonicMs();
    this.writeLock.lock();
    //获取entryLog个数
    final int entriesCount = request.getEntriesCount();
    try {
        //校验当前节点是否活跃
        if (!this.state.isActive()) {
            LOG.warn("Node {} is not in active state, currTerm={}.", getNodeId(), this.currTerm);
            return RpcResponseFactory.newResponse(RaftError.EINVAL, "Node %s is not in active state, state %s.",
                getNodeId(), this.state.name());
        }
        //校验传入的serverId是否能被正常解析
        final PeerId serverId = new PeerId();
        if (!serverId.parse(request.getServerId())) {
            LOG.warn("Node {} received AppendEntriesRequest from {} serverId bad format.", getNodeId(),
                request.getServerId());
            return RpcResponseFactory.newResponse(RaftError.EINVAL, "Parse serverId failed: %s.",
                request.getServerId());
        }
        //校验任期
        // Check stale term
        if (request.getTerm() < this.currTerm) {
            LOG.warn("Node {} ignore stale AppendEntriesRequest from {}, term={}, currTerm={}.", getNodeId(),
                request.getServerId(), request.getTerm(), this.currTerm);
            return AppendEntriesResponse.newBuilder() //
                .setSuccess(false) //
                .setTerm(this.currTerm) //
                .build();
        }

        // Check term and state to step down
        //当前节点如果不是Follower节点的话要执行StepDown操作
        checkStepDown(request.getTerm(), serverId);
        //这说明请求的节点不是当前节点的leader
        if (!serverId.equals(this.leaderId)) {
            LOG.error("Another peer {} declares that it is the leader at term {} which was occupied by leader {}.",
                serverId, this.currTerm, this.leaderId);
            // Increase the term by 1 and make both leaders step down to minimize the
            // loss of split brain
            stepDown(request.getTerm() + 1, false, new Status(RaftError.ELEADERCONFLICT,
                "More than one leader in the same term."));
            return AppendEntriesResponse.newBuilder() //
                .setSuccess(false) //
                .setTerm(request.getTerm() + 1) //
                .build();
        }

        updateLastLeaderTimestamp(Utils.monotonicMs());

        //校验是否正在生成快照
        if (entriesCount > 0 && this.snapshotExecutor != null && this.snapshotExecutor.isInstallingSnapshot()) {
            LOG.warn("Node {} received AppendEntriesRequest while installing snapshot.", getNodeId());
            return RpcResponseFactory.newResponse(RaftError.EBUSY, "Node %s:%s is installing snapshot.",
                this.groupId, this.serverId);
        }
        //传入的是发起请求节点的nextIndex-1
        final long prevLogIndex = request.getPrevLogIndex();
        final long prevLogTerm = request.getPrevLogTerm();
        final long localPrevLogTerm = this.logManager.getTerm(prevLogIndex);
        //发起请求的节点prevLogIndex对应的任期和当前节点的index所对应的任期不匹配
        if (localPrevLogTerm != prevLogTerm) {
            final long lastLogIndex = this.logManager.getLastLogIndex();

            LOG.warn(
                "Node {} reject term_unmatched AppendEntriesRequest from {}, term={}, prevLogIndex={}, prevLogTerm={}, localPrevLogTerm={}, lastLogIndex={}, entriesSize={}.",
                getNodeId(), request.getServerId(), request.getTerm(), prevLogIndex, prevLogTerm, localPrevLogTerm,
                lastLogIndex, entriesCount);

            return AppendEntriesResponse.newBuilder() //
                .setSuccess(false) //
                .setTerm(this.currTerm) //
                .setLastLogIndex(lastLogIndex) //
                .build();
        }
        //响应心跳或者发送的是sendEmptyEntry
        if (entriesCount == 0) {
            // heartbeat
            final AppendEntriesResponse.Builder respBuilder = AppendEntriesResponse.newBuilder() //
                .setSuccess(true) //
                .setTerm(this.currTerm)
                //  返回当前节点的最新的index
                .setLastLogIndex(this.logManager.getLastLogIndex());
            doUnlock = false;
            this.writeLock.unlock();
            // see the comments at FollowerStableClosure#run()
            this.ballotBox.setLastCommittedIndex(Math.min(request.getCommittedIndex(), prevLogIndex));
            return respBuilder.build();
        }

        // Parse request
        long index = prevLogIndex;
        final List<LogEntry> entries = new ArrayList<>(entriesCount);
        ByteBuffer allData = null;
        if (request.hasData()) {
            allData = request.getData().asReadOnlyByteBuffer();
        }
        //获取所有数据
        final List<RaftOutter.EntryMeta> entriesList = request.getEntriesList();
        for (int i = 0; i < entriesCount; i++) {
            final RaftOutter.EntryMeta entry = entriesList.get(i);
            index++;
            if (entry.getType() != EnumOutter.EntryType.ENTRY_TYPE_UNKNOWN) {
                //给logEntry属性设值
                final LogEntry logEntry = new LogEntry();
                logEntry.setId(new LogId(index, entry.getTerm()));
                logEntry.setType(entry.getType());
                if (entry.hasChecksum()) {
                    logEntry.setChecksum(entry.getChecksum()); // since 1.2.6
                }
                //将数据填充到logEntry
                final long dataLen = entry.getDataLen();
                if (dataLen > 0) {
                    final byte[] bs = new byte[(int) dataLen];
                    assert allData != null;
                    allData.get(bs, 0, bs.length);
                    logEntry.setData(ByteBuffer.wrap(bs));
                }

                if (entry.getPeersCount() > 0) {
                    //只有配置类型的entry才有多个Peer
                    if (entry.getType() != EnumOutter.EntryType.ENTRY_TYPE_CONFIGURATION) {
                        throw new IllegalStateException(
                                "Invalid log entry that contains peers but is not ENTRY_TYPE_CONFIGURATION type: "
                                        + entry.getType());
                    }

                    final List<PeerId> peers = new ArrayList<>(entry.getPeersCount());
                    for (final String peerStr : entry.getPeersList()) {
                        final PeerId peer = new PeerId();
                        peer.parse(peerStr);
                        peers.add(peer);
                    }
                    logEntry.setPeers(peers);

                    if (entry.getOldPeersCount() > 0) {
                        final List<PeerId> oldPeers = new ArrayList<>(entry.getOldPeersCount());
                        for (final String peerStr : entry.getOldPeersList()) {
                            final PeerId peer = new PeerId();
                            peer.parse(peerStr);
                            oldPeers.add(peer);
                        }
                        logEntry.setOldPeers(oldPeers);
                    }
                } else if (entry.getType() == EnumOutter.EntryType.ENTRY_TYPE_CONFIGURATION) {
                    throw new IllegalStateException(
                            "Invalid log entry that contains zero peers but is ENTRY_TYPE_CONFIGURATION type");
                }

                // Validate checksum
                if (this.raftOptions.isEnableLogEntryChecksum() && logEntry.isCorrupted()) {
                    long realChecksum = logEntry.checksum();
                    LOG.error(
                            "Corrupted log entry received from leader, index={}, term={}, expectedChecksum={}, " +
                             "realChecksum={}",
                            logEntry.getId().getIndex(), logEntry.getId().getTerm(), logEntry.getChecksum(),
                            realChecksum);
                    return RpcResponseFactory.newResponse(RaftError.EINVAL,
                            "The log entry is corrupted, index=%d, term=%d, expectedChecksum=%d, realChecksum=%d",
                            logEntry.getId().getIndex(), logEntry.getId().getTerm(), logEntry.getChecksum(),
                            realChecksum);
                }

                entries.add(logEntry);
            }
        }
        //存储日志，并回调返回response
        final FollowerStableClosure closure = new FollowerStableClosure(request, AppendEntriesResponse.newBuilder()
            .setTerm(this.currTerm), this, done, this.currTerm);
        this.logManager.appendEntries(entries, closure);
        // update configuration after _log_manager updated its memory status
        this.conf = this.logManager.checkAndSetConfiguration(this.conf);
        return null;
    } finally {
        if (doUnlock) {
            this.writeLock.unlock();
        }
        this.metrics.recordLatency("handle-append-entries", Utils.monotonicMs() - startMs);
        this.metrics.recordSize("handle-append-entries-count", entriesCount);
    }
}
```

handleAppendEntriesRequest方法写的很长，但是实际上做了很多校验的事情，具体的处理逻辑不多

1. 校验当前的Node节点是否还处于活跃状态，如果不是的话，那么直接返回一个error的response
2. 校验请求的serverId的格式是否正确，不正确则返回一个error的response
3. 校验请求的任期是否小于当前的任期，如果是那么返回一个AppendEntriesResponse类型的response
4. 调用checkStepDown方法检测当前节点的任期，以及状态，是否有leader等
5. 如果请求的serverId和当前节点的leaderId是不是同一个，用来校验是不是leader发起的请求，如果不是返回一个AppendEntriesResponse
6. 校验是否正在生成快照
7. 获取请求的Index在当前节点中对应的LogEntry的任期是不是和请求传入的任期相同，不同的话则返回AppendEntriesResponse
8. 如果传入的entriesCount为零，那么leader发送的可能是心跳或者发送的是sendEmptyEntry，返回AppendEntriesResponse，并将当前任期和最新index封装返回
9. 请求的数据不为空，那么遍历所有的数据
10. 实例化一个logEntry，并且将数据和属性设置到logEntry实例中，最后将logEntry放入到entries集合中
11. 调用logManager将数据批量提交日志写入 RocksDB

### 发送响应给leader[#](https://www.cnblogs.com/luozhiyun/p/12005975.html#2460071594)

最终发送给leader的响应是通过AppendEntriesRequestProcessor的sendSequenceResponse来发送的

```java
Copyvoid sendSequenceResponse(final String groupId, final String peerId, final int seq,
                          final AsyncContext asyncContext, final BizContext bizContext, final Message msg) {
    final Connection connection = bizContext.getConnection();
    //获取context，维度是groupId和peerId
    final PeerRequestContext ctx = getPeerRequestContext(groupId, peerId, connection);
    final PriorityQueue<SequenceMessage> respQueue = ctx.responseQueue;
    assert (respQueue != null);

    synchronized (Utils.withLockObject(respQueue)) {
        //将要响应的数据放入到优先队列中
        respQueue.add(new SequenceMessage(asyncContext, msg, seq));
        //校验队列里面的数据是否超过了256
        if (!ctx.hasTooManyPendingResponses()) {
            while (!respQueue.isEmpty()) {
                final SequenceMessage queuedPipelinedResponse = respQueue.peek();
                //如果序列对应不上，那么就不发送响应
                if (queuedPipelinedResponse.sequence != getNextRequiredSequence(groupId, peerId, connection)) {
                    // sequence mismatch, waiting for next response.
                    break;
                }
                respQueue.remove();
                try {
                    //发送响应
                    queuedPipelinedResponse.sendResponse();
                } finally {
                    //序列加一
                    getAndIncrementNextRequiredSequence(groupId, peerId, connection);
                }
            }
        } else {
            LOG.warn("Closed connection to peer {}/{}, because of too many pending responses, queued={}, max={}",
                ctx.groupId, peerId, respQueue.size(), ctx.maxPendingResponses);
            connection.close();
            // Close the connection if there are too many pending responses in queue.
            removePeerRequestContext(groupId, peerId);
        }
    }
}
```

这个方法会将要发送的数据依次压入到PriorityQueue优先队列中进行排序，然后获取序列号最小的元素和nextRequiredSequence比较，如果不相等，那么则是出现了乱序的情况，那么就不发送请求

## Leader处理日志复制的Response[#](https://www.cnblogs.com/luozhiyun/p/12005975.html#369472526)

Leader收到Follower发过来的Response响应之后会调用Replicator的onRpcReturned方法

```java
Copystatic void onRpcReturned(final ThreadId id, final RequestType reqType, final Status status, final Message request,
                          final Message response, final int seq, final int stateVersion, final long rpcSendTime) {
    if (id == null) {
        return;
    }
    final long startTimeMs = Utils.nowMs();
    Replicator r;
    if ((r = (Replicator) id.lock()) == null) {
        return;
    }
    //检查版本号，因为每次resetInflights都会让version加一，所以检查一下
    if (stateVersion != r.version) {
        LOG.debug(
            "Replicator {} ignored old version response {}, current version is {}, request is {}\n, and response is {}\n, status is {}.",
            r, stateVersion, r.version, request, response, status);
        id.unlock();
        return;
    }
    //使用优先队列按seq排序,最小的会在第一个
    final PriorityQueue<RpcResponse> holdingQueue = r.pendingResponses;
    //这里用一个优先队列是因为响应是异步的，seq小的可能响应比seq大慢
    holdingQueue.add(new RpcResponse(reqType, seq, status, request, response, rpcSendTime));
    //默认holdingQueue队列里面的数量不能超过256
    if (holdingQueue.size() > r.raftOptions.getMaxReplicatorInflightMsgs()) {
        LOG.warn("Too many pending responses {} for replicator {}, maxReplicatorInflightMsgs={}",
            holdingQueue.size(), r.options.getPeerId(), r.raftOptions.getMaxReplicatorInflightMsgs());
        //重新发送探针
        //清空数据
        r.resetInflights();
        r.state = State.Probe;
        r.sendEmptyEntries(false);
        return;
    }

    boolean continueSendEntries = false;

    final boolean isLogDebugEnabled = LOG.isDebugEnabled();
    StringBuilder sb = null;
    if (isLogDebugEnabled) {
        sb = new StringBuilder("Replicator ").append(r).append(" is processing RPC responses,");
    }
    try {
        int processed = 0;
        while (!holdingQueue.isEmpty()) {
            //取出holdingQueue里seq最小的数据
            final RpcResponse queuedPipelinedResponse = holdingQueue.peek();

            //如果Follower没有响应的话就会出现次序对不上的情况，那么就不往下走了
            //sequence mismatch, waiting for next response.
            if (queuedPipelinedResponse.seq != r.requiredNextSeq) {
                // 如果之前存在处理，则到此直接break循环
                if (processed > 0) {
                    if (isLogDebugEnabled) {
                        sb.append("has processed ").append(processed).append(" responses,");
                    }
                    break;
                } else {
                    //Do not processed any responses, UNLOCK id and return.
                    continueSendEntries = false;
                    id.unlock();
                    return;
                }
            }
            //走到这里说明seq对的上，那么就移除优先队列里面seq最小的数据
            holdingQueue.remove();
            processed++;
            //获取inflights队列里的第一个元素
            final Inflight inflight = r.pollInflight();
            //发起一个请求的时候会将inflight放入到队列中
            //如果为空，那么就忽略
            if (inflight == null) {
                // The previous in-flight requests were cleared.
                if (isLogDebugEnabled) {
                    sb.append("ignore response because request not found:").append(queuedPipelinedResponse)
                        .append(",\n");
                }
                continue;
            }
            //seq没有对上，说明顺序乱了，重置状态
            if (inflight.seq != queuedPipelinedResponse.seq) {
                // reset state
                LOG.warn(
                    "Replicator {} response sequence out of order, expect {}, but it is {}, reset state to try again.",
                    r, inflight.seq, queuedPipelinedResponse.seq);
                r.resetInflights();
                r.state = State.Probe;
                continueSendEntries = false;
                // 锁住节点，根据错误类别等待一段时间
                r.block(Utils.nowMs(), RaftError.EREQUEST.getNumber());
                return;
            }
            try {
                switch (queuedPipelinedResponse.requestType) {
                    case AppendEntries:
                        //处理日志复制的response
                        continueSendEntries = onAppendEntriesReturned(id, inflight, queuedPipelinedResponse.status,
                            (AppendEntriesRequest) queuedPipelinedResponse.request,
                            (AppendEntriesResponse) queuedPipelinedResponse.response, rpcSendTime, startTimeMs, r);
                        break;
                    case Snapshot:
                        //处理快照的response
                        continueSendEntries = onInstallSnapshotReturned(id, r, queuedPipelinedResponse.status,
                            (InstallSnapshotRequest) queuedPipelinedResponse.request,
                            (InstallSnapshotResponse) queuedPipelinedResponse.response);
                        break;
                }
            } finally {
                if (continueSendEntries) {
                    // Success, increase the response sequence.
                    r.getAndIncrementRequiredNextSeq();
                } else {
                    // The id is already unlocked in onAppendEntriesReturned/onInstallSnapshotReturned, we SHOULD break out.
                    break;
                }
            }
        }
    } finally {
        if (isLogDebugEnabled) {
            sb.append(", after processed, continue to send entries: ").append(continueSendEntries);
            LOG.debug(sb.toString());
        }
        if (continueSendEntries) {
            // unlock in sendEntries.
            r.sendEntries();
        }
    }
}
```

1. 检查版本号，因为每次resetInflights都会让version加一，所以检查一下是不是同一批的数据
2. 获取Replicator的pendingResponses队列，然后将当前响应的数据封装成RpcResponse实例加入到队列中
3. 校验队列里面的元素是否大于256，大于256则清空数据重新同步
4. 校验holdingQueue队列里面的seq最小的序列数据序列和当前的requiredNextSeq是否相同，不同的话如果是刚进入循环那么直接break退出循环
5. 获取inflights队列中第一个元素，如果seq没有对上，说明顺序乱了，重置状态
6. 调用onAppendEntriesReturned方法处理日志复制的response
7. 如果处理成功，那么则调用sendEntries继续发送复制日志到Follower

**Replicator#onAppendEntriesReturned**

```java
Copyprivate static boolean onAppendEntriesReturned(final ThreadId id, final Inflight inflight, final Status status,
                                               final AppendEntriesRequest request,
                                               final AppendEntriesResponse response, final long rpcSendTime,
                                               final long startTimeMs, final Replicator r) {
    //校验数据序列有没有错
    if (inflight.startIndex != request.getPrevLogIndex() + 1) {
        LOG.warn(
            "Replicator {} received invalid AppendEntriesResponse, in-flight startIndex={}, request prevLogIndex={}, reset the replicator state and probe again.",
            r, inflight.startIndex, request.getPrevLogIndex());
        r.resetInflights();
        r.state = State.Probe;
        // unlock id in sendEmptyEntries
        r.sendEmptyEntries(false);
        return false;
    }
    //度量
    // record metrics
    if (request.getEntriesCount() > 0) {
        r.nodeMetrics.recordLatency("replicate-entries", Utils.monotonicMs() - rpcSendTime);
        r.nodeMetrics.recordSize("replicate-entries-count", request.getEntriesCount());
        r.nodeMetrics.recordSize("replicate-entries-bytes", request.getData() != null ? request.getData().size()
            : 0);
    }

    final boolean isLogDebugEnabled = LOG.isDebugEnabled();
    StringBuilder sb = null;
    if (isLogDebugEnabled) {
        sb = new StringBuilder("Node "). //
            append(r.options.getGroupId()).append(":").append(r.options.getServerId()). //
            append(" received AppendEntriesResponse from "). //
            append(r.options.getPeerId()). //
            append(" prevLogIndex=").append(request.getPrevLogIndex()). //
            append(" prevLogTerm=").append(request.getPrevLogTerm()). //
            append(" count=").append(request.getEntriesCount());
    }
    //如果follower因为崩溃，RPC调用失败等原因没有收到成功响应
    //那么需要阻塞一段时间再进行调用
    if (!status.isOk()) {
        // If the follower crashes, any RPC to the follower fails immediately,
        // so we need to block the follower for a while instead of looping until
        // it comes back or be removed
        // dummy_id is unlock in block
        if (isLogDebugEnabled) {
            sb.append(" fail, sleep.");
            LOG.debug(sb.toString());
        }
        //如果注册了Replicator状态监听器，那么通知所有监听器
        notifyReplicatorStatusListener(r, ReplicatorEvent.ERROR, status);
        if (++r.consecutiveErrorTimes % 10 == 0) {
            LOG.warn("Fail to issue RPC to {}, consecutiveErrorTimes={}, error={}", r.options.getPeerId(),
                r.consecutiveErrorTimes, status);
        }
        r.resetInflights();
        r.state = State.Probe;
        // unlock in in block
        r.block(startTimeMs, status.getCode());
        return false;
    }
    r.consecutiveErrorTimes = 0;
    //响应失败
    if (!response.getSuccess()) {
        // Leader 的切换，表明可能出现过一次网络分区，从新跟随新的 Leader
        if (response.getTerm() > r.options.getTerm()) {
            if (isLogDebugEnabled) {
                sb.append(" fail, greater term ").append(response.getTerm()).append(" expect term ")
                    .append(r.options.getTerm());
                LOG.debug(sb.toString());
            }
            // 获取当前本节点的表示对象——NodeImpl
            final NodeImpl node = r.options.getNode();
            r.notifyOnCaughtUp(RaftError.EPERM.getNumber(), true);
            r.destroy();
            // 调整自己的 term 任期值
            node.increaseTermTo(response.getTerm(), new Status(RaftError.EHIGHERTERMRESPONSE,
                "Leader receives higher term heartbeat_response from peer:%s", r.options.getPeerId()));
            return false;
        }
        if (isLogDebugEnabled) {
            sb.append(" fail, find nextIndex remote lastLogIndex ").append(response.getLastLogIndex())
                .append(" local nextIndex ").append(r.nextIndex);
            LOG.debug(sb.toString());
        }
        if (rpcSendTime > r.lastRpcSendTimestamp) {
            r.lastRpcSendTimestamp = rpcSendTime;
        }
        // Fail, reset the state to try again from nextIndex.
        r.resetInflights();
        //如果Follower最新的index小于下次要发送的index，那么设置为Follower响应的index
        // prev_log_index and prev_log_term doesn't match
        if (response.getLastLogIndex() + 1 < r.nextIndex) {
            LOG.debug("LastLogIndex at peer={} is {}", r.options.getPeerId(), response.getLastLogIndex());
            // The peer contains less logs than leader
            r.nextIndex = response.getLastLogIndex() + 1;
        } else {
            // The peer contains logs from old term which should be truncated,
            // decrease _last_log_at_peer by one to test the right index to keep
            if (r.nextIndex > 1) {
                LOG.debug("logIndex={} dismatch", r.nextIndex);
                r.nextIndex--;
            } else {
                LOG.error("Peer={} declares that log at index=0 doesn't match, which is not supposed to happen",
                    r.options.getPeerId());
            }
        }
        //响应失败需要重新获取Follower的日志信息，用来重新同步
        // dummy_id is unlock in _send_heartbeat
        r.sendEmptyEntries(false);
        return false;
    }
    if (isLogDebugEnabled) {
        sb.append(", success");
        LOG.debug(sb.toString());
    }
    // success
    //响应成功检查任期
    if (response.getTerm() != r.options.getTerm()) {
        r.resetInflights();
        r.state = State.Probe;
        LOG.error("Fail, response term {} dismatch, expect term {}", response.getTerm(), r.options.getTerm());
        id.unlock();
        return false;
    }
    if (rpcSendTime > r.lastRpcSendTimestamp) {
        r.lastRpcSendTimestamp = rpcSendTime;
    }
    // 本次提交的日志数量
    final int entriesSize = request.getEntriesCount();
    if (entriesSize > 0) {
        // 节点确认提交
        r.options.getBallotBox().commitAt(r.nextIndex, r.nextIndex + entriesSize - 1, r.options.getPeerId());
        if (LOG.isDebugEnabled()) {
            LOG.debug("Replicated logs in [{}, {}] to peer {}", r.nextIndex, r.nextIndex + entriesSize - 1,
                r.options.getPeerId());
        }
    } else {
        // The request is probe request, change the state into Replicate.
        r.state = State.Replicate;
    }
    r.nextIndex += entriesSize;
    r.hasSucceeded = true;
    r.notifyOnCaughtUp(RaftError.SUCCESS.getNumber(), false);
    // dummy_id is unlock in _send_entries
    if (r.timeoutNowIndex > 0 && r.timeoutNowIndex < r.nextIndex) {
        r.sendTimeoutNow(false, false);
    }
    return true;
}
```

onAppendEntriesReturned方法也非常的长，但是我们要有点耐心往下看

1. 校验数据序列有没有错
2. 进行度量和拼接日志操作
3. 判断一下返回的状态如果不是正常的，那么就通知监听器，进行重置操作并阻塞一定时间后再发送
4. 如果返回Success状态为false，那么校验一下任期，因为Leader 的切换，表明可能出现过一次网络分区，需要重新跟随新的 Leader；如果任期没有问题那么就进行重置操作,并根据Follower返回的最新的index来重新设值nextIndex
5. 如果各种校验都没有问题的话，那么进行日志提交确认，更新最新的日志提交位置索引