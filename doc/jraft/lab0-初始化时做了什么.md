我们这次依然用上次的例子CounterServer来进行讲解：

我这里就不贴整个代码了

```java
Copypublic static void main(final String[] args) throws IOException {
    if (args.length != 4) {
        System.out
            .println("Useage : java com.alipay.sofa.jraft.example.counter.CounterServer {dataPath} {groupId} {serverId} {initConf}");
        System.out
            .println("Example: java com.alipay.sofa.jraft.example.counter.CounterServer " +
                    "/tmp/server1 " +
                    "counter " +
                    "127.0.0.1:8081 127.0.0.1:8081,127.0.0.1:8082,127.0.0.1:8083");
        System.exit(1);
    }
    //日志存储的路径
    final String dataPath = args[0];
    //SOFAJRaft集群的名字
    final String groupId = args[1];
    //当前节点的ip和端口
    final String serverIdStr = args[2];
    //集群节点的ip和端口
    final String initConfStr = args[3];

    final NodeOptions nodeOptions = new NodeOptions();
    // 为了测试,调整 snapshot 间隔等参数
    // 设置选举超时时间为 1 秒
    nodeOptions.setElectionTimeoutMs(1000);
    // 关闭 CLI 服务。
    nodeOptions.setDisableCli(false);
    // 每隔30秒做一次 snapshot
    nodeOptions.setSnapshotIntervalSecs(30);
    // 解析参数
    final PeerId serverId = new PeerId();
    if (!serverId.parse(serverIdStr)) {
        throw new IllegalArgumentException("Fail to parse serverId:" + serverIdStr);
    }
    final Configuration initConf = new Configuration();
    //将raft分组加入到Configuration的peers数组中
    if (!initConf.parse(initConfStr)) {
        throw new IllegalArgumentException("Fail to parse initConf:" + initConfStr);
    }
    // 设置初始集群配置
    nodeOptions.setInitialConf(initConf);

    // 启动
    final CounterServer counterServer = new CounterServer(dataPath, groupId, serverId, nodeOptions);
    System.out.println("Started counter server at port:"
                       + counterServer.getNode().getNodeId().getPeerId().getPort());
}
```

我们在启动server的main方法的时候会传入日志存储的路径、SOFAJRaft集群的名字、当前节点的ip和端口、集群节点的ip和端口并设值到NodeOptions中，作为当前节点启动的参数。

这里会将当前节点初始化为一个PeerId对象
**PeerId**

```java
Copy//存放当前节点的ip和端口号
private Endpoint            endpoint         = new Endpoint(Utils.IP_ANY, 0);

//默认是0
private int                 idx; 
//是一个ip：端口的字符串
private String              str;
public PeerId() {
    super();
}

public boolean parse(final String s) {
    final String[] tmps = StringUtils.split(s, ':');
    if (tmps.length != 3 && tmps.length != 2) {
        return false;
    }
    try {
        final int port = Integer.parseInt(tmps[1]);
        this.endpoint = new Endpoint(tmps[0], port);
        if (tmps.length == 3) {
            this.idx = Integer.parseInt(tmps[2]);
        } else {
            this.idx = 0;
        }
        this.str = null;
        return true;
    } catch (final Exception e) {
        LOG.error("Parse peer from string failed: {}", s, e);
        return false;
    }
}
```

PeerId的parse方法会将传入的ip：端口解析之后对变量进行一些赋值的操作。

然后会调用到CounterServer的构造器中：
**CounterServer**

```java
Copypublic CounterServer(final String dataPath, final String groupId, final PeerId serverId,
                     final NodeOptions nodeOptions) throws IOException {
    // 初始化路径
    FileUtils.forceMkdir(new File(dataPath));

    // 这里让 raft RPC 和业务 RPC 使用同一个 RPC server, 通常也可以分开
    final RpcServer rpcServer = new RpcServer(serverId.getPort());
    RaftRpcServerFactory.addRaftRequestProcessors(rpcServer);
    // 注册业务处理器
    rpcServer.registerUserProcessor(new GetValueRequestProcessor(this));
    rpcServer.registerUserProcessor(new IncrementAndGetRequestProcessor(this));
    // 初始化状态机
    this.fsm = new CounterStateMachine();
    // 设置状态机到启动参数
    nodeOptions.setFsm(this.fsm);
    // 设置存储路径
    // 日志, 必须
    nodeOptions.setLogUri(dataPath + File.separator + "log");
    // 元信息, 必须
    nodeOptions.setRaftMetaUri(dataPath + File.separator + "raft_meta");
    // snapshot, 可选, 一般都推荐
    nodeOptions.setSnapshotUri(dataPath + File.separator + "snapshot");
    // 初始化 raft group 服务框架
    this.raftGroupService = new RaftGroupService(groupId, serverId, nodeOptions, rpcServer);
    // 启动
    this.node = this.raftGroupService.start();
}
```

这个方法主要是调用NodeOptions的各种方法进行设置，然后调用raftGroupService的start方法启动raft节点。

### RaftGroupService[#](https://www.cnblogs.com/luozhiyun/p/11651414.html#3350796144)

我们来到RaftGroupService的start方法：
**RaftGroupService#start**

```java
Copypublic synchronized Node start(final boolean startRpcServer) {
    //如果已经启动了，那么就返回
    if (this.started) {
        return this.node;
    }
    //校验serverId和groupId
    if (this.serverId == null || this.serverId.getEndpoint() == null
            || this.serverId.getEndpoint().equals(new Endpoint(Utils.IP_ANY, 0))) {
        throw new IllegalArgumentException("Blank serverId:" + this.serverId);
    }
    if (StringUtils.isBlank(this.groupId)) {
        throw new IllegalArgumentException("Blank group id" + this.groupId);
    }
    //Adds RPC server to Server.
    //设置当前node的ip和端口
    NodeManager.getInstance().addAddress(this.serverId.getEndpoint());

    //创建node
    this.node = RaftServiceFactory.createAndInitRaftNode(this.groupId, this.serverId, this.nodeOptions);
    if (startRpcServer) {
        //启动远程服务
        this.rpcServer.start();
    } else {
        LOG.warn("RPC server is not started in RaftGroupService.");
    }
    this.started = true;
    LOG.info("Start the RaftGroupService successfully.");
    return this.node;
}
```

这个方法会在一开始的时候对RaftGroupService在构造器实例化的参数进行校验，然后把当前节点的Endpoint添加到NodeManager的addrSet变量中，接着调用RaftServiceFactory#createAndInitRaftNode实例化Node节点。

每个节点都会启动一个rpc的服务，因为每个节点既可以被选举也可以投票给其他节点，节点之间需要互相通信，所以需要启动一个rpc服务。

**RaftServiceFactory#createAndInitRaftNode**

```java
Copypublic static Node createAndInitRaftNode(final String groupId, final PeerId serverId, final NodeOptions opts) {
    //实例化一个node节点
    final Node ret = createRaftNode(groupId, serverId);
    //为node节点初始化
    if (!ret.init(opts)) {
        throw new IllegalStateException("Fail to init node, please see the logs to find the reason.");
    }
    return ret;
}

public static Node createRaftNode(final String groupId, final PeerId serverId) {
    return new NodeImpl(groupId, serverId);
}
```

createAndInitRaftNode方法首先调用createRaftNode实例化一个Node的实例NodeImpl，然后调用其init方法进行初始化，主要的配置都是在init方法中完成的。

**NodeImpl**

```java
Copypublic NodeImpl(final String groupId, final PeerId serverId) {
    super();
    if (groupId != null) {
        //检验groupId是否符合格式规范
        Utils.verifyGroupId(groupId);
    }
    this.groupId = groupId;
    this.serverId = serverId != null ? serverId.copy() : null;
    //一开始的设置为未初始化
    this.state = State.STATE_UNINITIALIZED;
    //设置新的任期为0
    this.currTerm = 0;
    //设置最新的时间戳
    updateLastLeaderTimestamp(Utils.monotonicMs());
    this.confCtx = new ConfigurationCtx(this);
    this.wakingCandidate = null; 
    final int num = GLOBAL_NUM_NODES.incrementAndGet();
    LOG.info("The number of active nodes increment to {}.", num);
}
```

NodeImpl会在构造器中初始化一些参数。

### Node的初始化[#](https://www.cnblogs.com/luozhiyun/p/11651414.html#1793682028)

Node节点的所有的重要的配置都是在init方法中完成的，NodeImpl的init方法比较长所以分成代码块来进行讲解。

**NodeImpl#init**

```java
Copy//非空校验
Requires.requireNonNull(opts, "Null node options");
Requires.requireNonNull(opts.getRaftOptions(), "Null raft options");
Requires.requireNonNull(opts.getServiceFactory(), "Null jraft service factory");
//目前就一个实现：DefaultJRaftServiceFactory
this.serviceFactory = opts.getServiceFactory();
this.options = opts;
this.raftOptions = opts.getRaftOptions();
//基于 Metrics 类库的性能指标统计，具有丰富的性能统计指标，默认不开启度量工具
this.metrics = new NodeMetrics(opts.isEnableMetrics());

if (this.serverId.getIp().equals(Utils.IP_ANY)) {
    LOG.error("Node can't started from IP_ANY.");
    return false;
}

if (!NodeManager.getInstance().serverExists(this.serverId.getEndpoint())) {
    LOG.error("No RPC server attached to, did you forget to call addService?");
    return false;
}
//定时任务管理器
this.timerManager = new TimerManager();
//初始化定时任务管理器的内置线程池
if (!this.timerManager.init(this.options.getTimerPoolSize())) {
    LOG.error("Fail to init timer manager.");
    return false;
}

//定时任务管理器
this.timerManager = new TimerManager();
//初始化定时任务管理器的内置线程池
if (!this.timerManager.init(this.options.getTimerPoolSize())) {
    LOG.error("Fail to init timer manager.");
    return false;
}
```

这段代码主要是给各个变量赋值，然后进行校验判断一下serverId不能为0.0.0.0，当前的Endpoint必须要在NodeManager里面设置过等等（NodeManager的设置是在RaftGroupService的start方法里）。

然后会初始化一个全局的的定时调度管理器TimerManager：
**TimerManager**

```java
Copyprivate ScheduledExecutorService executor;

@Override
public boolean init(Integer coreSize) {
    this.executor = Executors.newScheduledThreadPool(coreSize, new NamedThreadFactory(
        "JRaft-Node-ScheduleThreadPool-", true));
    return true;
}
```

TimerManager的init方法就是初始化一个线程池，如果当前的服务器的cpu线程数*3 大于20 ，那么这个线程池的coreSize就是20，否则就是cpu线程数*3。

往下走是计时器的初始化：

```java
Copy// Init timers
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
//快照计时器
this.snapshotTimer = new RepeatedTimer("JRaft-SnapshotTimer", this.options.getSnapshotIntervalSecs() * 1000) {

    @Override
    protected void onTrigger() {
        handleSnapshotTimeout();
    }
};
```

voteTimer是用来控制选举的，如果选举超时，当前的节点又是候选者角色，那么就会发起选举。
electionTimer是预投票计时器。候选者在发起投票之前，先发起预投票，如果没有得到半数以上节点的反馈，则候选者就会识趣的放弃参选。
stepDownTimer定时检查是否需要重新选举leader。当前的leader可能出现它的Follower可能并没有整个集群的1/2却还没有下台的情况，那么这个时候会定期的检查看leader的Follower是否有那么多，没有那么多的话会强制让leader下台。
snapshotTimer快照计时器。这个计时器会每隔1小时触发一次生成一个快照。

这些计时器的具体实现现在暂时不表，等到要讲具体功能的时候再进行梳理。

这些计时器有一个共同的特点就是会根据不同的计时器返回一个在一定范围内随机的时间。返回一个随机的时间可以防止多个节点在同一时间内同时发起投票选举从而降低选举失败的概率。

继续往下看：

```java
Copythis.configManager = new ConfigurationManager();
//初始化一个disruptor，采用多生产者模式
this.applyDisruptor = DisruptorBuilder.<LogEntryAndClosure>newInstance() //
        //设置disruptor大小，默认16384
        .setRingBufferSize(this.raftOptions.getDisruptorBufferSize()) //
        .setEventFactory(new LogEntryAndClosureFactory()) //
        .setThreadFactory(new NamedThreadFactory("JRaft-NodeImpl-Disruptor-", true)) //
        .setProducerType(ProducerType.MULTI) //
        .setWaitStrategy(new BlockingWaitStrategy()) //
        .build();
//设置事件处理器
this.applyDisruptor.handleEventsWith(new LogEntryAndClosureHandler());
//设置异常处理器
this.applyDisruptor.setDefaultExceptionHandler(new LogExceptionHandler<Object>(getClass().getSimpleName()));
// 启动disruptor的线程
this.applyQueue = this.applyDisruptor.start();
//如果开启了metrics统计
if (this.metrics.getMetricRegistry() != null) {
    this.metrics.getMetricRegistry().register("jraft-node-impl-disruptor",
            new DisruptorMetricSet(this.applyQueue));
}
```

这里初始化了一个Disruptor作为消费队列，不清楚Disruptor的朋友可以去看我上一篇文章：[Disruptor—核心概念及体验](https://www.cnblogs.com/luozhiyun/p/11631305.html)。然后还校验了metrics是否开启，默认是不开启的。

继续往下看：

```java
Copy//fsmCaller封装对业务 StateMachine 的状态转换的调用以及日志的写入等
this.fsmCaller = new FSMCallerImpl();
//初始化日志存储功能
if (!initLogStorage()) {
    LOG.error("Node {} initLogStorage failed.", getNodeId());
    return false;
}
//初始化元数据存储功能
if (!initMetaStorage()) {
    LOG.error("Node {} initMetaStorage failed.", getNodeId());
    return false;
}
//对FSMCaller初始化
if (!initFSMCaller(new LogId(0, 0))) {
    LOG.error("Node {} initFSMCaller failed.", getNodeId());
    return false;
}
//实例化投票箱
this.ballotBox = new BallotBox();
final BallotBoxOptions ballotBoxOpts = new BallotBoxOptions();
ballotBoxOpts.setWaiter(this.fsmCaller);
ballotBoxOpts.setClosureQueue(this.closureQueue);
//初始化ballotBox的属性
if (!this.ballotBox.init(ballotBoxOpts)) {
    LOG.error("Node {} init ballotBox failed.", getNodeId());
    return false;
}
//初始化快照存储功能
if (!initSnapshotStorage()) {
    LOG.error("Node {} initSnapshotStorage failed.", getNodeId());
    return false;
}
//校验日志文件索引的一致性
final Status st = this.logManager.checkConsistency();
if (!st.isOk()) {
    LOG.error("Node {} is initialized with inconsistent log, status={}.", getNodeId(), st);
    return false;
}
//配置管理raft group中的信息
this.conf = new ConfigurationEntry();
this.conf.setId(new LogId());
// if have log using conf in log, else using conf in options
if (this.logManager.getLastLogIndex() > 0) {
    this.conf = this.logManager.checkAndSetConfiguration(this.conf);
} else {
    this.conf.setConf(this.options.getInitialConf());
}
```

这段代码主要是对快照、日志、元数据等功能初始化。

```java
Copythis.replicatorGroup = new ReplicatorGroupImpl();
//收其他节点或者客户端发过来的请求，转交给对应服务处理
this.rpcService = new BoltRaftClientService(this.replicatorGroup);
final ReplicatorGroupOptions rgOpts = new ReplicatorGroupOptions();
rgOpts.setHeartbeatTimeoutMs(heartbeatTimeout(this.options.getElectionTimeoutMs()));
rgOpts.setElectionTimeoutMs(this.options.getElectionTimeoutMs());
rgOpts.setLogManager(this.logManager);
rgOpts.setBallotBox(this.ballotBox);
rgOpts.setNode(this);
rgOpts.setRaftRpcClientService(this.rpcService);
rgOpts.setSnapshotStorage(this.snapshotExecutor != null ? this.snapshotExecutor.getSnapshotStorage() : null);
rgOpts.setRaftOptions(this.raftOptions);
rgOpts.setTimerManager(this.timerManager);

// Adds metric registry to RPC service.
this.options.setMetricRegistry(this.metrics.getMetricRegistry());
//初始化rpc服务
if (!this.rpcService.init(this.options)) {
    LOG.error("Fail to init rpc service.");
    return false;
}
this.replicatorGroup.init(new NodeId(this.groupId, this.serverId), rgOpts);

this.readOnlyService = new ReadOnlyServiceImpl();
final ReadOnlyServiceOptions rosOpts = new ReadOnlyServiceOptions();
rosOpts.setFsmCaller(this.fsmCaller);
rosOpts.setNode(this);
rosOpts.setRaftOptions(this.raftOptions);
//只读服务初始化
if (!this.readOnlyService.init(rosOpts)) {
    LOG.error("Fail to init readOnlyService.");
    return false;
}
```

这段代码主要是初始化replicatorGroup、rpcService以及readOnlyService。

接下来是最后一段的代码：

```java
Copy// set state to follower
this.state = State.STATE_FOLLOWER;

if (LOG.isInfoEnabled()) {
    LOG.info("Node {} init, term={}, lastLogId={}, conf={}, oldConf={}.", getNodeId(), this.currTerm,
            this.logManager.getLastLogId(false), this.conf.getConf(), this.conf.getOldConf());
}

//如果快照执行器不为空，并且生成快照的时间间隔大于0，那么就定时生成快照
if (this.snapshotExecutor != null && this.options.getSnapshotIntervalSecs() > 0) {
    LOG.debug("Node {} start snapshot timer, term={}.", getNodeId(), this.currTerm);
    this.snapshotTimer.start();
}

if (!this.conf.isEmpty()) {
    //新启动的node需要重新选举
    stepDown(this.currTerm, false, new Status());
}

if (!NodeManager.getInstance().add(this)) {
    LOG.error("NodeManager add {} failed.", getNodeId());
    return false;
}

// Now the raft node is started , have to acquire the writeLock to avoid race
// conditions
this.writeLock.lock();
//这个分支表示当前的jraft集群里只有一个节点，那么个节点必定是leader直接进行选举就好了
if (this.conf.isStable() && this.conf.getConf().size() == 1 && this.conf.getConf().contains(this.serverId)) {
    // The group contains only this server which must be the LEADER, trigger
    // the timer immediately.
    electSelf();
} else {
    this.writeLock.unlock();
}

return true;
```

这段代码里会将当前的状态设置为Follower，然后启动快照定时器定时生成快照。
如果当前的集群不是单节点集群需要做一下stepDown，表示新生成的Node节点需要重新进行选举。
最下面有一个if分支，如果当前的jraft集群里只有一个节点，那么个节点必定是leader直接进行选举就好了，所以会直接调用electSelf进行选举。
选举的代码我们就暂时略过，要不然后面就没得讲了。

到这里整个NodeImpl实例的init方法就分析完了，这个方法很长，但是还是做了很多事情的