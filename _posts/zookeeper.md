---
title: zookeeper
tags:
 - zookeeper
categories: 经验分享
---
Google Chubby的开源实现
* Zab(Zookeeper atomic broadcast protocol): 一致性协议
* (a service for coordinating processes of distributed applications)：分布式协调服务

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
