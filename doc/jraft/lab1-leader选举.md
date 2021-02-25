## 开篇[#](https://www.cnblogs.com/luozhiyun/p/11743469.html#1902832508)

在上一篇文章当中，我们讲解了NodeImpl在init方法里面会初始化话的动作，选举也是在这个方法里面进行的，这篇文章来从这个方法里详细讲一下选举的过程。

由于我这里介绍的是如何实现的，所以请大家先看一下原理：[SOFAJRaft 选举机制剖析 | SOFAJRaft 实现原理](https://www.sofastack.tech/blog/sofa-jraft-election-mechanism/)

文章比较长，我也慢慢的写了半个月时间~

## jraft选举的大致思路

为了提升文章的可读性，我还是希望花一部分篇幅讲清楚选举机制的基本原理，以便后面集中注意力于代码实现。下面是一段图文比喻，帮助理解的同时也让整篇文章不至于过早陷入细节的讨论。



### **问题1：选举要解决什么** 



一个分布式集群可以看成是由多条战船组成的一支舰队，各船之间通过旗语来保持信息交流。这样的一支舰队中，各船既不会互相完全隔离，但也没法像陆地上那样保持非常密切的联系，天气、海况、船距、船只战损情况导致船舰之间的联系存在但不可靠。

舰队作为一个统一的作战集群，需要有统一的共识、步调一致的命令，这些都要依赖于旗舰指挥。各舰船要服从于旗舰发出的指令，当旗舰不能继续工作后，需要有别的战舰接替旗舰的角色。

![图片](https://mmbiz.qpic.cn/mmbiz_png/nibOZpaQKw09QtJ4Q6Hic5brTPcliayb6gtDmiaSCGLdrUw3HDhIzp3MqzejK7oTBBuV0vqIunfgELvq1HM8MnznvQ/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)
图1 - 所有船有责任随时准备接替旗舰

如何在舰队中，选出一艘得到大家认可的旗舰，这就是 SOFAJRaft 中选举要解决的问题。



### **问题2：何时可以发起选举**



何时可以发起选举？换句话说，触发选举的标准是什么？这个标准必须对所有战舰一致，这样就能够在标准得到满足时，所有舰船公平的参与竞选。在 SOFAJRaft 中，触发标准就是**通信超时**，当旗舰在规定的一段时间内没有与 Follower 舰船进行通信时，Follower 就可以认为旗舰已经不能正常担任旗舰的职责，则 Follower 可以去尝试接替旗舰的角色。这段通信超时被称为 Election Timeout （简称 **ET**）， Follower 接替旗舰的尝试也就是发起选举请求。

![img](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)
图2 - ET 触发其他船竞选旗舰



### **问题3：何时真正发起选举** 



在选举中，只有当舰队中超过一半的船都同意，发起选举的船才能够成为旗舰，否则就只能开始一轮新的选举。所以如果 Follower 采取尽快发起选举的策略，试图尽早为舰队选出可用的旗舰，就可能引发一个潜在的风险：可能多艘船几乎同时发起选举，结果其中任何一支船都没能获得超过半数选票，导致这一轮选举无果，然后失败的 Follower 们再一次几乎同时发起选举，又一次失败，再选举 again，再失败 again ···

![图片](https://mmbiz.qpic.cn/mmbiz_png/nibOZpaQKw09QtJ4Q6Hic5brTPcliayb6gtL8BtrIticS5Eiade2djr9kPaZGAbpR8sibbCSs5BEIdjmXYy3N9pjOPAQ/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)
图3 - 同时发起选举，选票被瓜分

为避免这种情况，我们采用随机的选举触发时间，当 Follower 发现旗舰失联之后，会选择等待一段随机的时间 Random(0, ET) ，如果等待期间没有选出旗舰，则 Follower 再发起选举。

![图片](https://mmbiz.qpic.cn/mmbiz_png/nibOZpaQKw09QtJ4Q6Hic5brTPcliayb6gtY3bgibhQibZTwVchDia3XNxibNWbicpQUGOcaJzbHhzKKBibQJ5JKMmHWIeg/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)
图4 - 随机等待时间



### **问题4：哪些候选者值得选票** 



SOFAJRaft 的选举中包含了对两个属性的判断：LogIndex 和 Term，这是整个选举算法的核心部分，也是容易让人产生困惑的地方，因此我们做一下解释：

1. Term
   我们会对舰队中旗舰的历史进行编号，比如舰队的第1任旗舰、第2任旗舰，这个数字我们就用 Term 来表示。由于舰队中同时最多只能有一艘舰船担任旗舰，所以每一个 Term 只归属于一艘舰船，显然 Term 是单调递增的。
2. LogIndex
   每任旗舰在职期间都会发布一些指令（称其为“旗舰令”，类比“总统令”），这些旗舰令当然也是要编号归档的，这个编号我们用 Term 和 LogIndex 两个维度来标识，表示“第 Term 任旗舰发布的第 LogIndex 号旗舰令”。不同于现实中的总统令，我们的旗舰令中的 LogIndex 是一直递增的，不会因为旗舰的更迭而从头开始计算。

![img](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)
图5 - 总统令 Vs 旗舰令，LogIndex 稍有区别

所有的舰船都尽可能保存了过去从旗舰接收到的旗舰令，所以我们选举的标准就是哪艘船保存了最完整的旗舰令，那他就最有资格接任旗舰。具体来说，参与投票的船 V 不会对下面两种候选者 C 投票：一种是 lastTermC < lastTermV；另一种是 (lastTermV == lastTermC) && (lastLogIndexV > lastLogIndexC)。

稍作解释，第一种情况说明候选者 C 最后一次通信过的旗舰已经不是最新的旗舰了，至少比 V 更滞后，所以它所知道的旗舰令也不可能比 V 更完整。第二种情况说明，虽然 C 和 V 都与同一个旗舰有过通信，但是候选者 C 从旗舰处获得的旗舰令不如 V 完整 (lastLogIndexV > lastLogIndexC)，所以 V 不会投票给它。

![图片](https://mmbiz.qpic.cn/mmbiz_png/nibOZpaQKw09QtJ4Q6Hic5brTPcliayb6gtUDzxn3onaCAWbZTAbgPGXuxLCgx9ibBnRd5eIRB46V4IFnLUZARxC2Q/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)
图6 - Follower 船 b 拒绝了船 c 而投票给船 a，船 a 旗舰令有一个空白框表示“第 Term 任旗舰”没有发布过任何旗舰令



### **问题5：如何避免不够格的候选者“捣乱”**



如上一小节所说，SOFAJRaft 将 LogIndex 和 Term 作为选举的评选标准，所以当一艘船发起选举之前，会自增 Term 然后填到选举请求里发给其他船只 （可能是一段很复杂的旗语），表示自己竞选“第 Term + 1 任”旗舰。

这里要先说明一个机制，它被用来保证各船只的 Term 同步递增：当参与投票的 Follower 船收到这个投票请求后，如果发现自己的 Term 比投票请求里的小，就会自觉更新自己的 Term 向候选者看齐，这样能够很方便的将 Term 递增的信息同步到整个舰队中。

![图片](https://mmbiz.qpic.cn/mmbiz_png/nibOZpaQKw09QtJ4Q6Hic5brTPcliayb6gtpbCcLsmR8hrVQQCIyaGZEtjqS3CFCY0H8gS5xee9GBTHCkd0BZhPvA/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)
图7 - Follower 船根据投票请求更新自己的 Term

但是这种机制也带来一个麻烦，如果一艘船因为自己的原因没有看到旗舰发出的旗语，他就会自以为是的试图竞选成为新的旗舰，虽然不断发起选举且一直未能当选（因为旗舰和其他船都正常通信），但是它却通过自己的投票请求实际抬升了全局的 Term，这在 SOFAJRaft 算法中会迫使旗舰 stepdown （从旗舰的位置上退下来）。

![图片](https://mmbiz.qpic.cn/mmbiz_png/nibOZpaQKw09QtJ4Q6Hic5brTPcliayb6gtOw29wKicUAqXY0IycGWWjVC9usDD2V89FvoSNibrXgxswGiabjY1ogNVA/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)
图8 - 自以为是的捣乱者，迫使旗舰 stepdown

所以我们需要一种机制阻止这种“捣乱”，这就是预投票 (pre-vote) 环节。候选者在发起投票之前，先发起预投票，如果没有得到半数以上节点的反馈，则候选者就会识趣的放弃参选，也就不会抬升全局的 Term。

![图片](https://mmbiz.qpic.cn/mmbiz_png/nibOZpaQKw09QtJ4Q6Hic5brTPcliayb6gtFlAfiaxDiaavRft9Yv7spFpUoQ9KAvItNPMCFahpGBDRXUiaC9WgicSaDQ/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)
图9 - Pre-vote 预投票



### **选举剖析**



在上面的比喻中，我们可以看到整个选举操作的主线任务就是：

1. Candidate 被 ET 触发
2. Candidate 开始尝试发起 pre-vote 预投票
3. Follower 判断是否认可该 pre-vote request
4. Candidate 根据 pre-vote response 来决定是否发起 RequestVoteRequest
5. Follower 判断是否认可该 RequestVoteRequest
6. Candidate 根据 response 来判断自己是否当选

这个过程可用下图表示：

![img](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)
图10 - 一次成功的选举

在代码层面，主要是由四个方法来处理这个流程：

```
com.alipay.sofa.jraft.core.NodeImpl#preVote //预投票com.alipay.sofa.jraft.core.NodeImpl#electSelf //投票com.alipay.sofa.jraft.core.NodeImpl#handlePreVoteRequest //处理预投票请求com.alipay.sofa.jraft.core.NodeImpl#handleRequestVoteRequest //处理投票请求
```

代码逻辑比较直观，所以我们用流程图来简述各个方法中的处理。

### 预投票和投票

![图片](https://mmbiz.qpic.cn/mmbiz_png/nibOZpaQKw09QtJ4Q6Hic5brTPcliayb6gtVzmO0QnBS8ugqKiae2jgLBdkiagFsFicqz2TSHzVh0hOxXXP6WVnjbtDA/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)
图11 - 预投票 Vs 投票

图中可见，预投票请求 preVote 和投票请求 electSelf 的流程基本类似，只是有几个细节不太一样：

1. preVote 是由超时触发；
2. preVote 在组装 Request 的时候将 term 赋值为 currTerm + 1，而 electSelf 是先将 term ++；
3. preVote 成功后，进入 electSelf，electSelf 成功后 become Leader。

### 处理请求

处理预投票和投票请求的逻辑也比较类似，同样用图来表示。

![图片](https://mmbiz.qpic.cn/mmbiz_png/nibOZpaQKw09QtJ4Q6Hic5brTPcliayb6gtkHtGibhxFxFplW7yf87UHhBuJ1BwS6lKoZDTiaibavW5w9tgpyg7ZDUUQ/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)
图12 - 处理预投票请求

![图片](https://mmbiz.qpic.cn/mmbiz_png/nibOZpaQKw09QtJ4Q6Hic5brTPcliayb6gtQ7ibiaLyFBpckWkgY8bR60ANbccPeuGgrLed1ByRctxKPEQSbLWvxic8g/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)
图13 - 处理投票请求

图中可见，处理两种请求的流程也基本类似，只是处理投票请求的时候，会有 stepdown 机制，强制使 Leader 从其 Leader 的身份退到 Follower。在具体的实现中，Leader 会通过租约的机制来避免一些没有必要的 stepdown，关于租约机制，可以参见之前的系列文章《[SOFAJRaft 线性一致读实现剖析 | SOFAJRaft 实现原理](http://mp.weixin.qq.com/s?__biz=MzUzMzU5Mjc1Nw==&mid=2247485218&idx=1&sn=17b0db520c2b2c04b947e1eccb704ca1&chksm=faa0e8f8cdd761ee7e4ad1f6765cf91d3aa5f65c797ca2660674ce0c6aafbaeab52db37256e5&scene=21#wechat_redirect)》。

## 选举过程分析[#](https://www.cnblogs.com/luozhiyun/p/11743469.html#1180661797)

我在这里只把有关选举的代码列举出来，其他的代码暂且忽略
**NodeImpl#init**

```java
Copypublic boolean init(final NodeOptions opts) {
		....
    // Init timers
    //设置投票计时器
    this.voteTimer = new RepeatedTimer("JRaft-VoteTimer", this.options.getElectionTimeoutMs()) {

        @Override
        protected void onTrigger() {
            //处理投票超时
            handleVoteTimeout();
        }

        @Override
        protected int adjustTimeout(final int timeoutMs) {
            //在一定范围内返回一个随机的时间戳
            return randomTimeout(timeoutMs);
        }
    };
    //设置预投票计时器
    //当leader在规定的一段时间内没有与 Follower 舰船进行通信时，
    // Follower 就可以认为leader已经不能正常担任旗舰的职责，则 Follower 可以去尝试接替leader的角色。
    // 这段通信超时被称为 Election Timeout
    //候选者在发起投票之前，先发起预投票
    this.electionTimer = new RepeatedTimer("JRaft-ElectionTimer", this.options.getElectionTimeoutMs()) {

        @Override
        protected void onTrigger() {
            handleElectionTimeout();
        }

        @Override
        protected int adjustTimeout(final int timeoutMs) {
            //在一定范围内返回一个随机的时间戳
            //为了避免同时发起选举而导致失败
            return randomTimeout(timeoutMs);
        }
    };
    //leader下台的计时器
    //定时检查是否需要重新选举leader
    this.stepDownTimer = new RepeatedTimer("JRaft-StepDownTimer", this.options.getElectionTimeoutMs() >> 1) {

        @Override
        protected void onTrigger() {
            handleStepDownTimeout();
        }
    };
		....
    if (!this.conf.isEmpty()) {
        //新启动的node需要重新选举
        stepDown(this.currTerm, false, new Status());
    }
		....
}
```

在这个init方法里面会初始化三个计时器是和选举有关的：

- voteTimer：这个timer负责定期的检查，如果当前的state的状态是候选者（STATE_CANDIDATE），那么就会发起选举
- electionTimer：在一定时间内如果leader没有与 Follower 进行通信时，Follower 就可以认为leader已经不能正常担任leader的职责，那么就会进行选举，在选举之前会先发起预投票，如果没有得到半数以上节点的反馈，则候选者就会识趣的放弃参选。所以这个timer负责预投票
- stepDownTimer：定时检查是否需要重新选举leader，如果当前的leader没有获得超过半数的Follower响应，那么这个leader就应该下台然后重新选举。

RepeatedTimer的分析我已经写好了：[2. SOFAJRaft源码分析—JRaft的定时任务调度器是怎么做的？](https://www.cnblogs.com/luozhiyun/p/11706171.html)

我们先跟着init方法的思路往下看，一般来说this.conf里面装的是整个集群的节点信息，是不会为空的，所以会调用stepDown，所以先从这个方法看起。

### leader下台[#](https://www.cnblogs.com/luozhiyun/p/11743469.html#464044989)

```java
Copyprivate void stepDown(final long term, final boolean wakeupCandidate, final Status status) {
    LOG.debug("Node {} stepDown, term={}, newTerm={}, wakeupCandidate={}.", getNodeId(), this.currTerm, term,
        wakeupCandidate);
    //校验一下当前节点的状态是否有异常，或正在关闭
    if (!this.state.isActive()) {
        return;
    }
    //如果是候选者，那么停止选举
    if (this.state == State.STATE_CANDIDATE) {
        //调用voteTimer的stop方法
        stopVoteTimer();
        //如果当前状态是leader或TRANSFERRING
    } else if (this.state.compareTo(State.STATE_TRANSFERRING) <= 0) {
        //让启动的stepDownTimer停止运作
        stopStepDownTimer();
        //清空选票箱中的内容
        this.ballotBox.clearPendingTasks();
        // signal fsm leader stop immediately
        if (this.state == State.STATE_LEADER) {
            //发送leader下台的事件给其他Follower
            onLeaderStop(status);
        }
    }
    // reset leader_id
    //重置当前节点的leader
    resetLeaderId(PeerId.emptyPeer(), status);

    // soft state in memory
    this.state = State.STATE_FOLLOWER;
    //重置Configuration的上下文
    this.confCtx.reset();
    updateLastLeaderTimestamp(Utils.monotonicMs());
    if (this.snapshotExecutor != null) {
        //停止当前的快照生成
        this.snapshotExecutor.interruptDownloadingSnapshots(term);
    }

    //设置任期为大的那个
    // meta state
    if (term > this.currTerm) {
        this.currTerm = term;
        this.votedId = PeerId.emptyPeer();
        //重设元数据信息保存到文件中
        this.metaStorage.setTermAndVotedFor(term, this.votedId);
    }

    if (wakeupCandidate) {
        this.wakingCandidate = this.replicatorGroup.stopAllAndFindTheNextCandidate(this.conf);
        if (this.wakingCandidate != null) {
            Replicator.sendTimeoutNowAndStop(this.wakingCandidate, this.options.getElectionTimeoutMs());
        }
    } else {
        //把replicatorGroup里面的所有replicator标记为stop
        this.replicatorGroup.stopAll();
    }
    //leader转移的时候会用到
    if (this.stopTransferArg != null) {
        if (this.transferTimer != null) {
            this.transferTimer.cancel(true);
        }
        // There is at most one StopTransferTimer at the same term, it's safe to
        // mark stopTransferArg to NULL
        this.stopTransferArg = null;
    }
    //启动
    this.electionTimer.start();
}
```

一个leader的下台需要做很多交接的工作：

1. 如果当前的节点是个候选人（STATE_CANDIDATE），那么这个时候会让它暂时不要投票
2. 如果当前的节点状态是（STATE_TRANSFERRING）表示正在转交leader或是leader（STATE_LEADER），那么就需要把当前节点的stepDownTimer这个定时器给关闭
3. 如果当前是leader（STATE_LEADER），那么就需要告诉状态机leader下台了，可以在状态机中对下台的动作做处理
4. 重置当前节点的leader，把当前节点的state状态设置为Follower，重置confCtx上下文
5. 停止当前的快照生成，设置新的任期，让所有的复制节点停止工作
6. 启动electionTimer

调用stopVoteTimer和stopStepDownTimer方法里面主要是调用相应的RepeatedTimer的stop方法，在stop方法里面会将stopped状态设置为ture，并将timeout设置为取消，并将这个timeout加入到cancelledTimeouts集合中去：
如果看了[2. SOFAJRaft源码分析—JRaft的定时任务调度器是怎么做的？](https://www.cnblogs.com/luozhiyun/p/11706171.html)这篇文章的话，那么下面这段代码应该一看就明白是怎么回事了的。

```java
Copypublic void stop() {
    this.lock.lock();
    try {
        if (this.stopped) {
            return;
        }
        this.stopped = true;
        if (this.timeout != null) {
            this.timeout.cancel();
            this.running = false;
            this.timeout = null;
        }
    } finally {
        this.lock.unlock();
    }
}
```

#### 状态机处理LEADER_STOP事件[#](https://www.cnblogs.com/luozhiyun/p/11743469.html#4261333613)

在调用NodeImpl的onLeaderStop方法中，实际上是调用了FSMCallerImpl的onLeaderStop方法
**NodeImpl#onLeaderStop**

```java
Copyprivate void onLeaderStop(final Status status) {
    this.replicatorGroup.clearFailureReplicators();
    this.fsmCaller.onLeaderStop(status);
}
```

**FSMCallerImpl#onLeaderStop**

```java
Copypublic boolean onLeaderStop(final Status status) {
    return enqueueTask((task, sequence) -> {
		  //设置当前task的状态为LEADER_STOP
        task.type = TaskType.LEADER_STOP;
        task.status = new Status(status);
    });
}

private boolean enqueueTask(final EventTranslator<ApplyTask> tpl) {
    if (this.shutdownLatch != null) {
        // Shutting down
        LOG.warn("FSMCaller is stopped, can not apply new task.");
        return false;
    }
    //使用Disruptor发布事件
    this.taskQueue.publishEvent(tpl);
    return true;
}
```

这个方法里像taskQueue队列里面发布了一个LEADER_STOP事件，taskQueue是在FSMCallerImpl的init方法中被初始化的：

```java
Copypublic boolean init(final FSMCallerOptions opts) {
    .....
    this.disruptor = DisruptorBuilder.<ApplyTask>newInstance() //
            .setEventFactory(new ApplyTaskFactory()) //
            .setRingBufferSize(opts.getDisruptorBufferSize()) //
            .setThreadFactory(new NamedThreadFactory("JRaft-FSMCaller-Disruptor-", true)) //
            .setProducerType(ProducerType.MULTI) //
            .setWaitStrategy(new BlockingWaitStrategy()) //
            .build();
    this.disruptor.handleEventsWith(new ApplyTaskHandler());
    this.disruptor.setDefaultExceptionHandler(new LogExceptionHandler<Object>(getClass().getSimpleName()));
    this.taskQueue = this.disruptor.start();
    .....
}
```

在taskQueue中发布了一个任务之后会交给ApplyTaskHandler进行处理

**ApplyTaskHandler**

```java
Copyprivate class ApplyTaskHandler implements EventHandler<ApplyTask> {
    // max committed index in current batch, reset to -1 every batch
    private long maxCommittedIndex = -1;

    @Override
    public void onEvent(final ApplyTask event, final long sequence, final boolean endOfBatch) throws Exception {
        this.maxCommittedIndex = runApplyTask(event, this.maxCommittedIndex, endOfBatch);
    }
}
```

每当有任务到达taskQueue队列的时候会调用ApplyTaskHandler的onEvent方法来处理事件，具体的执行逻辑由runApplyTask方法进行处理

**FSMCallerImpl#runApplyTask**

```java
Copyprivate long runApplyTask(final ApplyTask task, long maxCommittedIndex, final boolean endOfBatch) {
    CountDownLatch shutdown = null;
    ...
     switch (task.type) {
         ...
        case LEADER_STOP:
            this.currTask = TaskType.LEADER_STOP;
            doLeaderStop(task.status);
            break;
        ...
    }
		....
}
```

在runApplyTask方法里会对很多的事件进行处理，我们这里只看LEADER_STOP是怎么做的：

在switch里会调用doLeaderStop方法，这个方法会调用到FSMCallerImpl里面封装的StateMachine状态机的onLeaderStart方法：

```java
Copyprivate void doLeaderStop(final Status status) {
    this.fsm.onLeaderStop(status);
}
```

这样就可以对leader停止时进行客制化的处理了。

#### 重置leader[#](https://www.cnblogs.com/luozhiyun/p/11743469.html#914094110)

接下来会调用resetLeaderId(PeerId.emptyPeer(), status);方法来重置leader

```java
Copyprivate void resetLeaderId(final PeerId newLeaderId, final Status status) {
    if (newLeaderId.isEmpty()) {
        //这个判断表示如果当前节点是候选者或者是Follower，并且已经有leader了
        if (!this.leaderId.isEmpty() && this.state.compareTo(State.STATE_TRANSFERRING) > 0) {
            //向状态机装发布停止跟随该leader的事件
            this.fsmCaller.onStopFollowing(new LeaderChangeContext(this.leaderId.copy(), this.currTerm, status));
        }
        //把当前的leader设置为一个空值
        this.leaderId = PeerId.emptyPeer();
    } else {
        //如果当前节点没有leader
        if (this.leaderId == null || this.leaderId.isEmpty()) {
            //那么发布要跟随该leader的事件
            this.fsmCaller.onStartFollowing(new LeaderChangeContext(newLeaderId, this.currTerm, status));
        }
        this.leaderId = newLeaderId.copy();
    }
}
```

这个方法由两个作用，如果传入的newLeaderId不是个空的，那么就会设置一个新的leader，并向状态机发送一个START_FOLLOWING事件；如果传入的newLeaderId是空的，那么就会发送一个STOP_FOLLOWING事件，并把当前的leader置空。

#### 启动electionTimer，进行leader选举[#](https://www.cnblogs.com/luozhiyun/p/11743469.html#2433450650)

electionTimer是RepeatedTimer的实现类，在这里我就不多说了，上一篇文章已经介绍过了。

我这里来看看electionTimer的onTrigger方法是怎么处理选举事件的，electionTimer的onTrigger方法会调用NodeImpl的handleElectionTimeout方法，所以直接看这个方法：

**NodeImpl#handleElectionTimeout**

```java
Copyprivate void handleElectionTimeout() {
    boolean doUnlock = true;
    this.writeLock.lock();
    try {
        if (this.state != State.STATE_FOLLOWER) {
            return;
        }
        //如果当前选举没有超时则说明此轮选举有效
        if (isCurrentLeaderValid()) {
            return;
        }
        resetLeaderId(PeerId.emptyPeer(), new Status(RaftError.ERAFTTIMEDOUT, "Lost connection from leader %s.",
            this.leaderId));
        doUnlock = false;
        //预投票 (pre-vote) 环节
        //候选者在发起投票之前，先发起预投票，
        // 如果没有得到半数以上节点的反馈，则候选者就会识趣的放弃参选
        preVote();
    } finally {
        if (doUnlock) {
            this.writeLock.unlock();
        }
    }
}
```

在这个方法里，首先会加上一个写锁，然后进行校验，最后先发起一个预投票。

校验的时候会校验当前的状态是不是follower，校验leader和follower上次的通信时间是不是超过了ElectionTimeoutMs，如果没有超过，说明leader存活，没必要发起选举；如果通信超时，那么会将leader置空，然后调用预选举。

**NodeImpl#isCurrentLeaderValid**

```java
Copyprivate boolean isCurrentLeaderValid() {
    return Utils.monotonicMs() - this.lastLeaderTimestamp < this.options.getElectionTimeoutMs();
}
```

用当前时间和上次leader通信时间相减，如果小于ElectionTimeoutMs（默认1s），那么就没有超时，说明leader有效

### 预选票preVote[#](https://www.cnblogs.com/luozhiyun/p/11743469.html#974044018)

我们在handleElectionTimeout方法中最后调用了preVote方法，接下来重点看一下这个方法。

下面我将preVote拆分成几部分来进行讲解：
**NodeImpl#preVote part1**

```java
Copyprivate void preVote() {
    long oldTerm;
    try {
        LOG.info("Node {} term {} start preVote.", getNodeId(), this.currTerm);
        if (this.snapshotExecutor != null && this.snapshotExecutor.isInstallingSnapshot()) {
            LOG.warn(
                "Node {} term {} doesn't do preVote when installing snapshot as the configuration may be out of date.",
                getNodeId());
            return;
        }
        //conf里面记录了集群节点的信息，如果当前的节点不包含在集群里说明是由问题的
        if (!this.conf.contains(this.serverId)) {
            LOG.warn("Node {} can't do preVote as it is not in conf <{}>.", getNodeId(), this.conf);
            return;
        }
        //设置一下当前的任期
        oldTerm = this.currTerm;
    } finally {
        this.writeLock.unlock();
    } 
	  ....
}
```

这部分代码是一开始进到preVote这个方法首先要经过一些校验，例如当前的节点不能再安装快照的时候进行选举；查看一下当前的节点是不是在自己设置的conf里面，conf这个属性会包含了集群的所有节点；最后设置一下当前的任期后解锁。

**NodeImpl#preVote part2**

```java
Copyprivate void preVote() {
	  ....
    //返回最新的log实体类
    final LogId lastLogId = this.logManager.getLastLogId(true);

    boolean doUnlock = true;
    this.writeLock.lock();
    try {
        // pre_vote need defense ABA after unlock&writeLock
        //因为在上面没有重新加锁的间隙里可能会被别的线程改变了，所以这里校验一下
        if (oldTerm != this.currTerm) {
            LOG.warn("Node {} raise term {} when get lastLogId.", getNodeId(), this.currTerm);
            return;
        }
        //初始化预投票投票箱
        this.prevVoteCtx.init(this.conf.getConf(), this.conf.isStable() ? null : this.conf.getOldConf());
        for (final PeerId peer : this.conf.listPeers()) {
            //如果遍历的节点是当前节点就跳过
            if (peer.equals(this.serverId)) {
                continue;
            }
            //失联的节点也跳过
            if (!this.rpcService.connect(peer.getEndpoint())) {
                LOG.warn("Node {} channel init failed, address={}.", getNodeId(), peer.getEndpoint());
                continue;
            }
            //设置一个回调的类
            final OnPreVoteRpcDone done = new OnPreVoteRpcDone(peer, this.currTerm);
            //向被遍历到的这个节点发送一个预投票的请求
            done.request = RequestVoteRequest.newBuilder() //
                .setPreVote(true) // it's a pre-vote request.
                .setGroupId(this.groupId) //
                .setServerId(this.serverId.toString()) //
                .setPeerId(peer.toString()) //
                .setTerm(this.currTerm + 1) // next term，注意这里发送过去的任期会加一
                .setLastLogIndex(lastLogId.getIndex()) //
                .setLastLogTerm(lastLogId.getTerm()) //
                .build();
            this.rpcService.preVote(peer.getEndpoint(), done.request, done);
        }
        //自己也可以投给自己
        this.prevVoteCtx.grant(this.serverId);
        if (this.prevVoteCtx.isGranted()) {
            doUnlock = false;
            electSelf();
        }
    } finally {
        if (doUnlock) {
            this.writeLock.unlock();
        }
    }
}
```

这部分代码：

1. 首先会获取最新的log信息，由LogId封装，里面包含两部分，一部分是这个日志的index和写入这个日志所对应当时节点的一个term任期
2. 初始化预投票投票箱
3. 遍历所有的集群节点
4. 如果遍历的节点是当前节点就跳过，如果遍历的节点因为宕机或者手动下线等原因连接不上也跳过
5. 向遍历的节点发送一个RequestVoteRequest请求预投票给自己
6. 最后因为自己也是集群节点的一员，所以自己也投票给自己

#### 初始化预投票投票箱[#](https://www.cnblogs.com/luozhiyun/p/11743469.html#1663699012)

初始化预投票箱是调用了Ballot的init方法进行初始化，分别传入新的集群节点信息，和老的集群节点信息

```java
Copypublic boolean init(Configuration conf, Configuration oldConf) {
    this.peers.clear();
    this.oldPeers.clear();
    quorum = oldQuorum = 0;
    int index = 0;
    //初始化新的节点
    if (conf != null) {
        for (PeerId peer : conf) {
            this.peers.add(new UnfoundPeerId(peer, index++, false));
        }
    }
    //设置需要多少票数才能成为leader
    this.quorum = this.peers.size() / 2 + 1;
    ....
    return true;
}
```

我这里为了使逻辑更清晰，假设没有oldConf，省略oldConf相关设置。
这个方法里会遍历所有的peer节点，并将peer封装成UnfoundPeerId插入到peers集合中；然后设置quorum属性，这个属性会在每获得一个投票就减1，当减到0以下时说明获得了足够多的票数，就代表预投票成功。

#### 发起预投票请求[#](https://www.cnblogs.com/luozhiyun/p/11743469.html#2665565984)

```java
Copy//设置一个回调的类
final OnPreVoteRpcDone done = new OnPreVoteRpcDone(peer, this.currTerm);
//向被遍历到的这个节点发送一个预投票的请求
done.request = RequestVoteRequest.newBuilder() //
    .setPreVote(true) // it's a pre-vote request.
    .setGroupId(this.groupId) //
    .setServerId(this.serverId.toString()) //
    .setPeerId(peer.toString()) //
    .setTerm(this.currTerm + 1) // next term，注意这里发送过去的任期会加一
    .setLastLogIndex(lastLogId.getIndex()) //
    .setLastLogTerm(lastLogId.getTerm()) //
    .build();
this.rpcService.preVote(peer.getEndpoint(), done.request, done);
```

在构造RequestVoteRequest的时候，会将PreVote属性设置为true，表示这次请求是预投票；设置当前节点为ServerId；传给对方的任期是当前节点的任期加一。最后在发送成功收到响应之后会回调OnPreVoteRpcDone的run方法。

**OnPreVoteRpcDone#run**

```java
Copypublic void run(final Status status) {
    NodeImpl.this.metrics.recordLatency("pre-vote", Utils.monotonicMs() - this.startMs);
    if (!status.isOk()) {
        LOG.warn("Node {} PreVote to {} error: {}.", getNodeId(), this.peer, status);
    } else {
        handlePreVoteResponse(this.peer, this.term, getResponse());
    }
}
```

在这个方法中如果收到正常的响应，那么会调用handlePreVoteResponse方法处理响应

**OnPreVoteRpcDone#handlePreVoteResponse**

```java
Copypublic void handlePreVoteResponse(final PeerId peerId, final long term, final RequestVoteResponse response) {
    boolean doUnlock = true;
    this.writeLock.lock();
    try {
        //只有follower才可以尝试发起选举
        if (this.state != State.STATE_FOLLOWER) {
            LOG.warn("Node {} received invalid PreVoteResponse from {}, state not in STATE_FOLLOWER but {}.",
                getNodeId(), peerId, this.state);
            return;
        }
        
        if (term != this.currTerm) {
            LOG.warn("Node {} received invalid PreVoteResponse from {}, term={}, currTerm={}.", getNodeId(),
                peerId, term, this.currTerm);
            return;
        }
        //如果返回的任期大于当前的任期，那么这次请求也是无效的
        if (response.getTerm() > this.currTerm) {
            LOG.warn("Node {} received invalid PreVoteResponse from {}, term {}, expect={}.", getNodeId(), peerId,
                response.getTerm(), this.currTerm);
            stepDown(response.getTerm(), false, new Status(RaftError.EHIGHERTERMRESPONSE,
                "Raft node receives higher term pre_vote_response."));
            return;
        }
        LOG.info("Node {} received PreVoteResponse from {}, term={}, granted={}.", getNodeId(), peerId,
            response.getTerm(), response.getGranted());
        // check granted quorum?
        if (response.getGranted()) {
            this.prevVoteCtx.grant(peerId);
            //得到了半数以上的响应
            if (this.prevVoteCtx.isGranted()) {
                doUnlock = false;
                //进行选举
                electSelf();
            }
        }
    } finally {
        if (doUnlock) {
            this.writeLock.unlock();
        }
    }
}
```

这里做了3重校验，我们分别来谈谈：

1. 第一重试校验了当前的状态，如果不是FOLLOWER那么就不能发起选举。因为如果是leader节点，那么它不会选举，只能stepdown下台，把自己变成FOLLOWER后重新选举；如果是CANDIDATE，那么只能进行由FOLLOWER发起的投票，所以从功能上来说，只能FOLLOWER发起选举。
   从Raft 的设计上来说也只能由FOLLOWER来发起选举，所以这里进行了校验。
2. 第二重校验主要是校验发送请求时的任期和接受到响应时的任期还是不是一个，如果不是那么说明已经不是上次那轮的选举了，是一次失效的选举
3. 第三重校验是校验响应返回的任期是不是大于当前的任期，如果大于当前的任期，那么重置当前的leader

校验完之后响应的节点会返回一个授权，如果授权通过的话则调用Ballot的grant方法，表示给当前的节点投一票

**Ballot#grant**

```java
Copypublic void grant(PeerId peerId) {
    this.grant(peerId, new PosHint());
}

public PosHint grant(PeerId peerId, PosHint hint) {
    UnfoundPeerId peer = findPeer(peerId, peers, hint.pos0);
    if (peer != null) {
        if (!peer.found) {
            peer.found = true;
            this.quorum--;
        }
        hint.pos0 = peer.index;
    } else {
        hint.pos0 = -1;
    }
    .... 
    return hint;
}
```

grant方法会根据peerId去集群集合里面去找被封装的UnfoundPeerId实例，然后判断一下，如果没有被记录过，那么就将quorum减一，表示收到一票，然后将found设置为ture表示已经找过了。

在查找UnfoundPeerId实例的时候方法里面做了一个很有趣的设置：
首先在存入到peers集合里面的时候是这样的：

```java
Copyint index = 0;
for (PeerId peer : conf) {
    this.peers.add(new UnfoundPeerId(peer, index++, false));
}
```

这里会遍历conf，然后会存入index，index从零开始。
然后在查找的时候会传入peerId和posHint还有peers集合：

```java
Copyprivate UnfoundPeerId findPeer(PeerId peerId, List<UnfoundPeerId> peers, int posHint) {
    if (posHint < 0 || posHint >= peers.size() || !peers.get(posHint).peerId.equals(peerId)) {
        for (UnfoundPeerId ufp : peers) {
            if (ufp.peerId.equals(peerId)) {
                return ufp;
            }
        }
        return null;
    }

    return peers.get(posHint);
}
```

这里传入的posHint默认是-1 ，即如果是第一次传入，那么会遍历整个peers集合，然后一个个比对之后返回。

因为PosHint实例会在调用完之后将pos0设置为peer的index，如果grant方法不是第一次调用，那么在调用findPeer方法的时候就可以直接通过get方法获取，不用再遍历整个集合了。

这种写法也可以运用到平时的代码中去。

调用了grant方法之后会调用Ballot的isGranted判断一下是否达到了半数以上的响应。
**Ballot#isGranted**

```java
Copypublic boolean isGranted() {
    return this.quorum <= 0 && oldQuorum <= 0;
}
```

即判断一下投票箱里面的票数是不是被减到了0。如果返回是的话，那么就调用electSelf进行选举。

选举的方法暂时先不看，我们来看看预选举的请求是怎么被处理的

#### 响应RequestVoteRequest请求[#](https://www.cnblogs.com/luozhiyun/p/11743469.html#1658588091)

RequestVoteRequest请求的处理器是在RaftRpcServerFactory的addRaftRequestProcessors方法中被安置的，具体的处理时RequestVoteRequestProcessor。

具体的处理方法是交由processRequest0方法来处理的。

**RequestVoteRequestProcessor#processRequest0**

```java
Copypublic Message processRequest0(RaftServerService service, RequestVoteRequest request, RpcRequestClosure done) {
    //如果是预选举
    if (request.getPreVote()) {
        return service.handlePreVoteRequest(request);
    } else {
        return service.handleRequestVoteRequest(request);
    }
}
```

因为这个处理器可以处理选举和预选举，所以加了个判断。预选举的方法交给NodeImpl的handlePreVoteRequest来具体实现的。

**NodeImpl#handlePreVoteRequest**

```java
Copypublic Message handlePreVoteRequest(final RequestVoteRequest request) {
    boolean doUnlock = true;
    this.writeLock.lock();
    try {
        //校验这个节点是不是正常的节点
        if (!this.state.isActive()) {
            LOG.warn("Node {} is not in active state, currTerm={}.", getNodeId(), this.currTerm);
            return RpcResponseFactory.newResponse(RaftError.EINVAL, "Node %s is not in active state, state %s.",
                getNodeId(), this.state.name());
        }
        final PeerId candidateId = new PeerId();
        //发送过来的request请求携带的ServerId格式不能错
        if (!candidateId.parse(request.getServerId())) {
            LOG.warn("Node {} received PreVoteRequest from {} serverId bad format.", getNodeId(),
                request.getServerId());
            return RpcResponseFactory.newResponse(RaftError.EINVAL, "Parse candidateId failed: %s.",
                request.getServerId());
        }
        boolean granted = false;
        // noinspection ConstantConditions
        do {
            //已经有leader的情况
            if (this.leaderId != null && !this.leaderId.isEmpty() && isCurrentLeaderValid()) {
                LOG.info(
                    "Node {} ignore PreVoteRequest from {}, term={}, currTerm={}, because the leader {}'s lease is still valid.",
                    getNodeId(), request.getServerId(), request.getTerm(), this.currTerm, this.leaderId);
                break;
            }
            //请求的任期小于当前的任期
            if (request.getTerm() < this.currTerm) {
                LOG.info("Node {} ignore PreVoteRequest from {}, term={}, currTerm={}.", getNodeId(),
                    request.getServerId(), request.getTerm(), this.currTerm);
                // A follower replicator may not be started when this node become leader, so we must check it.
                  //那么这个节点也可能是leader，所以校验一下请求的节点是不是复制节点，重新加入到replicatorGroup中
                checkReplicator(candidateId);
                break;
            } else if (request.getTerm() == this.currTerm + 1) {
                // A follower replicator may not be started when this node become leader, so we must check it.
                // check replicator state
                //因为请求的任期和当前的任期相等，那么这个节点也可能是leader，
                 // 所以校验一下请求的节点是不是复制节点，重新加入到replicatorGroup中
                checkReplicator(candidateId);
            }
            doUnlock = false;
            this.writeLock.unlock();
            //获取最新的日志
            final LogId lastLogId = this.logManager.getLastLogId(true);

            doUnlock = true;
            this.writeLock.lock();
            final LogId requestLastLogId = new LogId(request.getLastLogIndex(), request.getLastLogTerm());
            //比较当前节点的日志完整度和请求节点的日志完整度
            granted = requestLastLogId.compareTo(lastLogId) >= 0;

            LOG.info(
                "Node {} received PreVoteRequest from {}, term={}, currTerm={}, granted={}, requestLastLogId={}, lastLogId={}.",
                getNodeId(), request.getServerId(), request.getTerm(), this.currTerm, granted, requestLastLogId,
                lastLogId);
        } while (false);//这个while蛮有意思，为了用break想尽了办法

        return RequestVoteResponse.newBuilder() //
            .setTerm(this.currTerm) //
            .setGranted(granted) //
            .build();
    } finally {
        if (doUnlock) {
            this.writeLock.unlock();
        }
    }
}
```

这个方法里面也是蛮有意思的，写的很长，但是逻辑很清楚。

1. 首先调用isActive，看一下当前节点是不是正常的节点，不是正常节点要返回Error信息
2. 将请求传过来的ServerId解析到candidateId实例中
3. 校验当前的节点如果有leader，并且leader有效的，那么就直接break，返回granted为false
4. 如果当前的任期大于请求的任期，那么调用checkReplicator检查自己是不是leader，如果是leader，那么将当前节点从failureReplicators移除，重新加入到replicatorMap中。然后直接break
5. 请求任期和当前任期相等的情况也要校验，只是不用break
6. 如果请求的日志比当前的最新的日志还要新，那么返回granted为true，代表授权成功

这里有一个有意思的地方是，因为java中只能在循环中goto，所以这里使用了do-while（false）只做单次的循环，这样就可以do代码块里使用break了。

下面稍微看一下checkReplicator：
**NodeImpl#checkReplicator**

```java
Copyprivate void checkReplicator(final PeerId candidateId) {
    if (this.state == State.STATE_LEADER) {
        this.replicatorGroup.checkReplicator(candidateId, false);
    }
}
```

这里判断一下是不是leader，然后就会调用ReplicatorGroupImpl的checkReplicator

**ReplicatorGroupImpl#checkReplicator**

```java
Copyprivate final ConcurrentMap<PeerId, ThreadId> replicatorMap      = new ConcurrentHashMap<>();

private final Set<PeerId>                     failureReplicators = new ConcurrentHashSet<>();

public void checkReplicator(final PeerId peer, final boolean lockNode) {
    //根据传入的peer获取相应的ThreadId
    final ThreadId rid = this.replicatorMap.get(peer);
    // noinspection StatementWithEmptyBody
    if (rid == null) {
        // Create replicator if it's not found for leader.
        final NodeImpl node = this.commonOptions.getNode();
        if (lockNode) {
            node.writeLock.lock();
        }
        try {
            //如果当前的节点是leader，并且传入的peer在failureReplicators中，那么重新添加到replicatorMap
            if (node.isLeader() && this.failureReplicators.contains(peer) && addReplicator(peer)) {
                this.failureReplicators.remove(peer);
            }
        } finally {
            if (lockNode) {
                node.writeLock.unlock();
            }
        }
    } else { // NOPMD
        // Unblock it right now.
        // Replicator.unBlockAndSendNow(rid);
    }
}
```

checkReplicator会从replicatorMap根据传入的peer实例校验一下是不是为空。因为replicatorMap里面存放了集群所有的节点。然后通过ReplicatorGroupImpl的commonOptions获取当前的Node实例，如果当前的node实例是leader，并且如果存在失败集合failureReplicators中的话就重新添加进replicatorMap中。

**ReplicatorGroupImpl#addReplicator**

```java
Copypublic boolean addReplicator(final PeerId peer) {
    //校验当前的任期
    Requires.requireTrue(this.commonOptions.getTerm() != 0);
    //如果replicatorMap里面已经有这个节点了，那么将它从failureReplicators集合中移除
    if (this.replicatorMap.containsKey(peer)) {
        this.failureReplicators.remove(peer);
        return true;
    }
    //赋值一个新的ReplicatorOptions
    final ReplicatorOptions opts = this.commonOptions == null ? new ReplicatorOptions() : this.commonOptions.copy();
    //新的ReplicatorOptions添加这个PeerId
    opts.setPeerId(peer);
    final ThreadId rid = Replicator.start(opts, this.raftOptions);
    if (rid == null) {
        LOG.error("Fail to start replicator to peer={}.", peer);
        this.failureReplicators.add(peer);
        return false;
    }
    return this.replicatorMap.put(peer, rid) == null;
}
```

addReplicator里面主要是做了两件事：1. 将要加入的节点从failureReplicators集合里移除；2. 将要加入的节点放入到replicatorMap集合中去。

### 投票electSelf[#](https://www.cnblogs.com/luozhiyun/p/11743469.html#860807944)

```java
Copyprivate void electSelf() {
    long oldTerm;
    try {
        LOG.info("Node {} start vote and grant vote self, term={}.", getNodeId(), this.currTerm);
        //1. 如果当前节点不在集群里面则不进行选举
        if (!this.conf.contains(this.serverId)) {
            LOG.warn("Node {} can't do electSelf as it is not in {}.", getNodeId(), this.conf);
            return;
        }
        //2. 大概是因为要进行正式选举了，把预选举关掉
        if (this.state == State.STATE_FOLLOWER) {
            LOG.debug("Node {} stop election timer, term={}.", getNodeId(), this.currTerm);
            this.electionTimer.stop();
        }
        //3. 清空leader
        resetLeaderId(PeerId.emptyPeer(), new Status(RaftError.ERAFTTIMEDOUT,
            "A follower's leader_id is reset to NULL as it begins to request_vote."));
        this.state = State.STATE_CANDIDATE;
        this.currTerm++;
        this.votedId = this.serverId.copy();
        LOG.debug("Node {} start vote timer, term={} .", getNodeId(), this.currTerm);
        //4. 开始发起投票定时器，因为可能投票失败需要循环发起投票
        this.voteTimer.start();
        //5. 初始化投票箱
        this.voteCtx.init(this.conf.getConf(), this.conf.isStable() ? null : this.conf.getOldConf());
        oldTerm = this.currTerm;
    } finally {
        this.writeLock.unlock();
    }

    final LogId lastLogId = this.logManager.getLastLogId(true);

    this.writeLock.lock();
    try {
        // vote need defense ABA after unlock&writeLock
        if (oldTerm != this.currTerm) {
            LOG.warn("Node {} raise term {} when getLastLogId.", getNodeId(), this.currTerm);
            return;
        }
        //6. 遍历所有节点
        for (final PeerId peer : this.conf.listPeers()) {
            if (peer.equals(this.serverId)) {
                continue;
            }
            if (!this.rpcService.connect(peer.getEndpoint())) {
                LOG.warn("Node {} channel init failed, address={}.", getNodeId(), peer.getEndpoint());
                continue;
            }
            final OnRequestVoteRpcDone done = new OnRequestVoteRpcDone(peer, this.currTerm, this);
            done.request = RequestVoteRequest.newBuilder() //
                .setPreVote(false) // It's not a pre-vote request.
                .setGroupId(this.groupId) //
                .setServerId(this.serverId.toString()) //
                .setPeerId(peer.toString()) //
                .setTerm(this.currTerm) //
                .setLastLogIndex(lastLogId.getIndex()) //
                .setLastLogTerm(lastLogId.getTerm()) //
                .build();
            this.rpcService.requestVote(peer.getEndpoint(), done.request, done);
        }

        this.metaStorage.setTermAndVotedFor(this.currTerm, this.serverId);
        this.voteCtx.grant(this.serverId);
        if (this.voteCtx.isGranted()) {
            //7. 投票成功，那么就晋升为leader
            becomeLeader();
        }
    } finally {
        this.writeLock.unlock();
    }
}
```

不要看这个方法这么长，其实都是和前面预选举的方法preVote重复度很高的。方法太长，所以标了号，从上面号码来一步步讲解：

1. 对当前的节点进行校验，如果当前节点不在集群里面则不进行选举
2. 因为是Follower发起的选举，所以大概是因为要进行正式选举了，把预选举定时器关掉
3. 清空leader再进行选举，注意这里会把votedId设置为当前节点，代表自己参选
4. 开始发起投票定时器，因为可能投票失败需要循环发起投票，voteTimer里面会根据当前的CANDIDATE状态调用electSelf进行选举
5. 调用init方法初始化投票箱，这里和prevVoteCtx是一样的
6. 遍历所有节点，然后向其他集群节点发送RequestVoteRequest请求，这里也是和preVote一样的，请求是被RequestVoteRequestProcessor处理器处理的。
7. 如果有超过半数以上的节点投票选中，那么就调用becomeLeader晋升为leader

我先来看看RequestVoteRequestProcessor怎么处理的选举：
在RequestVoteRequestProcessor的processRequest0会调用NodeImpl的handleRequestVoteRequest来处理具体的逻辑。

#### 处理投票请求[#](https://www.cnblogs.com/luozhiyun/p/11743469.html#4152224603)

**NodeImpl#handleRequestVoteRequest**

```java
Copypublic Message handleRequestVoteRequest(final RequestVoteRequest request) {
    boolean doUnlock = true;
    this.writeLock.lock();
    try {
        //是否存活
        if (!this.state.isActive()) {
            LOG.warn("Node {} is not in active state, currTerm={}.", getNodeId(), this.currTerm);
            return RpcResponseFactory.newResponse(RaftError.EINVAL, "Node %s is not in active state, state %s.",
                getNodeId(), this.state.name());
        }
        final PeerId candidateId = new PeerId();
        if (!candidateId.parse(request.getServerId())) {
            LOG.warn("Node {} received RequestVoteRequest from {} serverId bad format.", getNodeId(),
                request.getServerId());
            return RpcResponseFactory.newResponse(RaftError.EINVAL, "Parse candidateId failed: %s.",
                request.getServerId());
        }

        // noinspection ConstantConditions
        do {
            // check term
            if (request.getTerm() >= this.currTerm) {
                LOG.info("Node {} received RequestVoteRequest from {}, term={}, currTerm={}.", getNodeId(),
                        request.getServerId(), request.getTerm(), this.currTerm);
                //1. 如果请求的任期大于当前任期
                // increase current term, change state to follower
                if (request.getTerm() > this.currTerm) {
                    stepDown(request.getTerm(), false, new Status(RaftError.EHIGHERTERMRESPONSE,
                        "Raft node receives higher term RequestVoteRequest."));
                }
            } else {
                // ignore older term
                LOG.info("Node {} ignore RequestVoteRequest from {}, term={}, currTerm={}.", getNodeId(),
                    request.getServerId(), request.getTerm(), this.currTerm);
                break;
            }
            doUnlock = false;
            this.writeLock.unlock();

            final LogId lastLogId = this.logManager.getLastLogId(true);

            doUnlock = true;
            this.writeLock.lock();
            // vote need ABA check after unlock&writeLock
            if (request.getTerm() != this.currTerm) {
                LOG.warn("Node {} raise term {} when get lastLogId.", getNodeId(), this.currTerm);
                break;
            }
            //2. 判断日志完整度
            final boolean logIsOk = new LogId(request.getLastLogIndex(), request.getLastLogTerm())
                .compareTo(lastLogId) >= 0;
            //3. 判断当前的节点是不是已经投过票了
            if (logIsOk && (this.votedId == null || this.votedId.isEmpty())) {
                stepDown(request.getTerm(), false, new Status(RaftError.EVOTEFORCANDIDATE,
                    "Raft node votes for some candidate, step down to restart election_timer."));
                this.votedId = candidateId.copy();
                this.metaStorage.setVotedFor(candidateId);
            }
        } while (false);

        return RequestVoteResponse.newBuilder() //
            .setTerm(this.currTerm) //
            //4.同意投票的条件是当前的任期和请求的任期一样，并且已经将votedId设置为请求节点
            .setGranted(request.getTerm() == this.currTerm && candidateId.equals(this.votedId)) //
            .build();
    } finally {
        if (doUnlock) {
            this.writeLock.unlock();
        }
    }
}
```

这个方法大致也和handlePreVoteRequest差不多。我这里只分析一下我标注的。

1. 这里是判断当前的任期是小于请求的任期的，并且调用stepDown将请求任期设置为当前的任期，将当前的状态设置被Follower
2. 作为一个leader来做日志肯定是要比被请求的节点完整，所以这里判断一下日志是不是比被请求的节点日志完整
3. 如果日志是完整的，并且被请求的节点没有投票给其他的候选人，那么就将votedId设置为当前请求的节点
4. 给请求发送响应，同意投票的条件是当前的任期和请求的任期一样，并且已经将votedId设置为请求节点

#### 晋升leader[#](https://www.cnblogs.com/luozhiyun/p/11743469.html#1146075491)

投票完毕之后如果收到的票数大于一半，那么就会晋升为leader，调用becomeLeader方法。

```java
Copyprivate void becomeLeader() {
    Requires.requireTrue(this.state == State.STATE_CANDIDATE, "Illegal state: " + this.state);
    LOG.info("Node {} become leader of group, term={}, conf={}, oldConf={}.", getNodeId(), this.currTerm,
        this.conf.getConf(), this.conf.getOldConf());
    // cancel candidate vote timer
    //晋升leader之后就会把选举的定时器关闭了
    stopVoteTimer();
    //设置当前的状态为leader
    this.state = State.STATE_LEADER;
    this.leaderId = this.serverId.copy();
    //复制集群中设置新的任期
    this.replicatorGroup.resetTerm(this.currTerm);
    //遍历所有的集群节点
    for (final PeerId peer : this.conf.listPeers()) {
        if (peer.equals(this.serverId)) {
            continue;
        }
        LOG.debug("Node {} add replicator, term={}, peer={}.", getNodeId(), this.currTerm, peer);
        //如果成为leader，那么需要把自己的日志信息复制到其他节点
        if (!this.replicatorGroup.addReplicator(peer)) {
            LOG.error("Fail to add replicator, peer={}.", peer);
        }
    }
    // init commit manager
    this.ballotBox.resetPendingIndex(this.logManager.getLastLogIndex() + 1);
    // Register _conf_ctx to reject configuration changing before the first log
    // is committed.
    if (this.confCtx.isBusy()) {
        throw new IllegalStateException();
    }
    this.confCtx.flush(this.conf.getConf(), this.conf.getOldConf());
    //如果是leader了，那么就要定时的检查不是有资格胜任
    this.stepDownTimer.start();
}
```

这个方法里面首先会停止选举定时器，然后设置当前的状态为leader，并设值任期，然后遍历所有的节点将节点加入到复制集群中，最后将stepDownTimer打开，定时对leader进行校验是不是又半数以上的节点响应当前的leader。

好了，到这里就讲完了，希望下次还可以see you again~