---
title: zookeeper
tags:
 - zookeeper
categories: 经验分享
---
Google Chubby的开源实现
* Zab(Zookeeper atomic broadcast protocol): 一致性协议
* (a service for coordinating processes of distributed applications)：分布式协调服务

leader负责写服务和数据同步、follower提供读服务

paxos算法：Leader选举
Zab算法：接受leader数据更新

ZooKeeper is a centralized service for maintaining configuration information, naming, providing distributed synchronization, and providing group services.

### 配置管理
配置需要在分布式环境多个服务用到时，保证配置在集群中的一致性，使用实现了一致性协议(Zab)的服务。

应用：
* HBase 客户端连接Zookeeper，获得必要的HBase集群的配置信息。
* Kafka 使用Zookeeper来维护broker的信息
* Dubbo SOA框架 管理一些配置来实现服务治理

### 命名服务
对资源命名、通过名称定位唯一的资源。
* Java 命名和目录接口（Java Naming and Directory Interface，JNDI） J2EE规范，标准J2EE容器都提供了对JNDI规范的实现。
* 数据库中方案：自增ID(InnoDB 中AUTO_INCREMENT)(不能分布式) UUID(Universally Unique Identifier)全局唯一标识符(过长、没有明确语义)
* 分布式环境下自增、唯一ID

### 分布式锁
某个时刻只有一个服务工作、出问题时释放锁、转移到其他服务。 Leader Election(Leader选举)

### 集群管理
服务发现与资源分配
应用：
* Dubbo Zookeeper作为服务发现底层机制
* Kafka Zookeeper作为consumer上下线管理

## 配置
[zoo.cfg]
```
dataDir
dataLogDir
# zookeeper的持久化都存储在这两个目录 dataLogDir里是放到的顺序日志(WAL) dataDir里放的是内存数据结构的snapshot，便于快速恢复
  为了性能最大化建议分配到不同磁盘上，可充分利用磁盘顺序写的特性。
server.1=127.0.0.1:20881:30881
server.2=127.0.0.1:20882:30882
server.3=127.0.0.1:20883:30883
#server.后面为myid 前一个端口是Leader、Follower、Observer交换数据使用的。 后一个用于Zookeeper选举
tickTime=1000 #时间单位定量 1 tick表示1000ms 所有其他用到时间的地方都用多少tick表示
syncLimit=2 # 就表示Follower与Leader的心跳时间是2tick
maxClientCnxns #对于一个客户端连接数限制，默认60
maxSessionTimeout、minSessionTimeout 超过这个时间client没有与zookeeper server有联系，则这个session会被设置为过期。服务器可以设置这两个参数来限制客户端设置的范围
autopurge.snapRetainCount,autopurge.purgeInterval zookeeper自动删除与客户端交互过程中的日志 autopurge。purgeInterval设置多少小时清理一次 autopurge.snapRetainCount设置保留多少个snapshot，之前的则删除
```
myid

dataDir会放置一个myid 唯一标识这个服务 zookeeper会根据这个id来取出server.x上的配置
## 应用
* Hadoop 依靠zookeeper 实现hdfs自动故障转移、YARN ResourceManager的高可用性
* Hbase 使用zookeeper实现区域服务器的主选举(master election)、租赁管理以及区域服务器之间的其他通信
* Accumulo 构建于zookeeper之上的另一个排序分布式键/值存储。
* Solr 使用zookeeper实现领导者选举和集中式配置
* mesos 集群管理器 提供了分布式应用程序之间高效的资源隔离和共享。使用zookeeper实现了容错的、复制的主选举
## 启动过程
入口 org.apache.zookeeper.server.quorum.QuorumPeerMain
```
# 解析配置文件 启动清理日志
DatadirCleanupManager purgeMgr = new DatadirCleanupManager(config
                .getDataDir(), config.getDataLogDir(), config
                .getSnapRetainCount(), config.getPurgeInterval());
purgeMgr.start();
```
初始化ServerCnxnFactory 接受来自客户端的连接 启动tcp server。在Zookeeper里提供两种tcp server的实现

一个是使用java原生NIO的方式，另一个是使用Netty 默认是java NIO 一个典型的Reactor模型。
```
//首先根据配置创建对应factory的实例:NIOServerCnxnFactory 或者 NettyServerCnxnFactory
ServerCnxnFactory cnxnFactory = ServerCnxnFactory.createFactory();
//初始化配置
cnxnFactory.configure(config.getClientPortAddress(),config.getMaxClientCnxns());
```
创建几个SelectorThread处理数据读取和写出 先是创建ServerSocketChannel,bind等
```
this.ss =  ServerSocketChannel.open();
ss.socket().setReuseAddress(true);
ss.socket().bind(addr);
ss.configureBlocking(false);
```
创建一个AcceptThread线程来接收客户端的连接

初始化主要部分 首先创建一个QuorumPeer实例，这个类就是表示zookeeper集群中的一个节点，初始化QuorumPeer的时候有这么几个关键点:
1. 初始化FileTxnSnapLog,这个类主要管理Zookeeper中的操作日志(WAL)和snapshot。
2. 初始化ZKDatabase，这个类就是Zookeeper的目录结构在内存中的表示，所有的操作最后都会映射到这个类上面来。
3. 初始化决议validator(QuorumVerifier->QuorumMaj).这一步是从zoo.cfg的server.n这一部分初始化出集群的成员出来，有哪些需要参与投票(Follower)，有哪些只是observer.还有决定half是多少等。这一步每个节点会初始化一个QuorumServer对象，并且放到allMembers，votingMembers,observingMembers这几个map里。而且这里也对参与者的个数进行了一些判断
## leader选举
QuorumPeer的startLeaderElection方法包含leader选举逻辑

Zookeeper默认提供4种选举方式，默认第四种：FastLeaderElection

新集群 节点状态LOOKING，FOLLOWING，LEADING，OBSERVING

启动时都是LOOKING状态 如果参与选举但最后不是leader 则状态是FOLLOWING，如果不参与选举则是OBSERVING，leader的状态是LEADING

启动监听端口

在FastLeaderElection里有一个Manager的内部类，这个类里启动了两个线程:WorkerReceiver,WorkerSender。一个处理从别的节点接收消息的，一个是向外发送消息的。异步

run方法开始执行，调用选举算法开始选举
```
setCurrentVote(makeLEStrategy().lookForLeader());
```
lookForLeader中 更新逻辑时钟 每个节点选自己然后向其他节点广播这个选举信息-将选举信息放到由WorkerSender管理的一个队列里。
```
synchronized(this){
    //逻辑时钟           
    logicalclock++;
    //getInitLastLoggedZxid(), getPeerEpoch()这里先不关心是什么，后面会讨论
    updateProposal(getInitId(), getInitLastLoggedZxid(), getPeerEpoch());
}

//getInitId() 即是获取选谁，id就是myid里指定的那个数字，所以说一定要唯一
private long getInitId(){
        if(self.getQuorumVerifier().getVotingMembers().containsKey(self.getId()))       
            return self.getId();
        else return Long.MIN_VALUE;
}

//发送选举信息，异步发送
sendNotifications();
```
将选举信息投递 在WorkerSender里，WorkerSender从sendqueue里取出投票，交给QuorumCnxManager发送。 如果发送给自己的，则不发送进入接收队列
```
public void toSend(Long sid, ByteBuffer b) {
        if (self.getId() == sid) {
             b.position(0);
             addToRecvQueue(new Message(b.duplicate(), sid));
        } else {
             //发送给别的节点，判断之前是不是发送过
             if (!queueSendMap.containsKey(sid)) {
                 //这个SEND_CAPACITY的大小是1，所以如果之前已经有一个还在等待发送，则会把之前的一个删除掉，发送新的
                 ArrayBlockingQueue<ByteBuffer> bq = new ArrayBlockingQueue<ByteBuffer>(SEND_CAPACITY);
                 queueSendMap.put(sid, bq);
                 addToSendQueue(bq, b);

             } else {
                 ArrayBlockingQueue<ByteBuffer> bq = queueSendMap.get(sid);
                 if(bq != null){
                     addToSendQueue(bq, b);
                 } else {
                     LOG.error("No queue for server " + sid);
                 }
             }
             //这里是真正的发送逻辑了
             connectOne(sid);

        }
    }
```
connectOne是真正的发送，发送之前会把自己的id和选举地址发过去。然后判断要发送节点的id是否比自己的id大，如果大则不发送了

如果发送启动两个线程SendWorker,RecvWorker 发送时将发送出去的东西放到lastMessageSent的map里，如果queueSendMap里是空的，就发送lastMessageSent里的东西，确保对方一定收到

接收-receiveConnection方法 如果收到的信息的id比自己的id小，则断开连接，并尝试发送消息给这个id对应的节点(如果已经有SendWorker往这个节点发送数据，则不用)

如果接收到id比当前id大，则会有RecvWorker接收数据，RecvWorker会将接收到的放到recvQueue

FastLeaderElection的WorkerReceiver线程里会不断地从这个recvQueue里读取Message 处理一些协议上的事情，比如消息格式 还会看接受的消息是不是来自投票成员 如果是投票成员则会看看这个消息里的状态

如果是LOOKING状态并且当前的逻辑时钟比投票消息里的逻辑时钟要高，则会发通知告诉谁是leader
```
protected boolean totalOrderPredicate(long newId, long newZxid, long newEpoch, long curId, long curZxid, long curEpoch) {return ((newEpoch > curEpoch) ||
                ((newEpoch == curEpoch) &&
                ((newZxid > curZxid) || ((newZxid == curZxid) && (newId > curId)))));
    }
```
1. 判断消息里的epoch是不是比当前的大，如果大则消息里id对应的server就承认它是leader
2. 如果epoch相等则判断zxid，如果消息里的zxid大就承认它是leader
3. 如果前两个都相等比较server_id，如果消息里的server_id大就承认它是leader
前两个对于新集群是相等的

判断是否超过一半为leader
```
private boolean termPredicate(
            HashMap<Long, Vote> votes,
            Vote vote) {

        HashSet<Long> set = new HashSet<Long>();
        //遍历已经收到的投票集合，将等于当前投票的集合取出放到set中
        for (Map.Entry<Long,Vote> entry : votes.entrySet()) {
            if (self.getQuorumVerifier().getVotingMembers().containsKey(entry.getKey())
                    && vote.equals(entry.getValue())){
                set.add(entry.getKey());
            }
        }

        //统计set，也就是投某个id的票数是否超过一半
        return self.getQuorumVerifier().containsQuorum(set);
    }

    public boolean containsQuorum(Set<Long> ackSet) {
        return (ackSet.size() > half);
    }
```
如果选的是自己，则将自己的状态更新为LEADING,否则根据type 是FOLLOWING或OBSERVING

启动的时候就是根据zoo.cfg里的配置，向各个节点广播投票，一般都是选投自己。然后收到投票后就会进行进行判断。如果某个节点收到的投票数超过一半，那么它就是leader了

![spark](http://img.blog.csdn.net/20150815130507597)

## zookeeper client
1. create 在给定的path上创建节点，类似文件系统路径 类型：永久节点、永久顺序节点、临时节点、临时顺序节点。
          永久节点创建即保留，临时节点会话过期自动删除，这个特性用于集群感知，应用启动时将自己的ip地址作为临时节点下面，可通过这个特性感知服务的集群有哪些活着的
          顺序节点-在给定的path上加上一个顺序编号，是实现分布式锁的关键，共享资源时，在某个路径下创建临时顺序节点 顺序晓得获得锁、被释放后第二小的获得锁，以此可实现分布式公平锁
2. delete 删除给定节点 有这个version在分布式环境种我们用乐观锁确保一致性。先读取节点获取version然后删除
3. exists 返回Stat对象，如果给定的path不存在返回null 可提供Watcher对象。Watcher是事件处理器，一个回调。调用exists方法后，如果别人对这个path上的节点进行操作，比如创建、删除、或设置数据，Watcher会收到通知
4. setData/getData 设置/获取节点数据。getData也可以设置Watcher
5. getChildren 获取子节点，可以设置Watcher
6. sync zookeeper是一个集群，创建时半数以上的节点确认就认为是创建成功了。如果读取正好读取到一个落后的节点上，读取到旧的数据，这个时候执行sync操作，这个操作可以确保读取到最新的数据。     
```
查看zk服务器节点状态
./zkServer.sh status

连接zk集群客户端
./zkCli.sh -server 10.0.8.177:2181,10.0.8.178:2181,10.0.8.179:2181

查看当前zookeeper中包含的内容
ls /

创建新的znode，使用create /zk "myData" 创建了一个新的znode节点 "zk"以及与它关联的字符串
create /zk "myData"

get来确认znode是否包含创建的字符串
get /zk
set命令对zk所关联的字符串设置
set /zk "zsl"
删除znode
delete /zk
```
```
输出关于性能和连接的客户端列表
echo stat |nc 10.0.8.177 2181
输出相关服务配置的详细信息
echo conf |nc 10.0.8.177 2181

```

1. 三种角色：Leader Follower Observer .Leader和Follower参与投票 Observer只会听投票结果，不参与投票
2. 投票集群的节点数要求是奇数
3. 一个集群容忍挂掉的节点数等式为N=2F+1 N为投票集群节点数 F为能同时容忍失败节点数。3节点集群可挂1个
4. 一个写操作需要半数以上节点ack，所以集群节点数越多，整个集群可以抗挂点的节点数越多(越可靠)，但吞吐量越差
5. Zookeeper 里所有节点以及节点的数据都会放在内存里，形成一棵树的数据结构，并定时的dump snapshot到磁盘
6. Zookeeper的Client与Zookeeper之间维持的是长连接。并保持心跳，Client会与Zookeeper之间协商出一个Session超时时间出来(配置) 如果session超时时间内没有收到心跳，则该Session过期。
7. Client可以watch Zookeeper那个树形数据结构里的某个节点或数据，当有变化的时候就会得到通知

## 添加observer
```
peerType=observer
server.1:localhost:2181:3181:observer

```



## 运维
1. 最小生产集群 一般至少5个
2. 网络 考虑节点的网络结构 避免一台物理机、一个机柜、一个交换机
3. 分group 保护核心group leader+Follower为核心Group,核心group一般不向外提供服务，根据不同业务再加一些observer
