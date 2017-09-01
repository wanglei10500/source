---
title: hadoop
tags:
 - hadoop
categories: 经验分享
---

### hadoop MRv1局限
org.apache.hadoop.mapred包中，MRv1的Map和Reduce是通过接口实现，MRv1包括3个部分：
* 运行时环境(JobTracker和TaskTracker)
* 编程模型(MapReduce)
* 数据处理引擎(Map任务和Reduce任务)
不足:
* 可扩展性差：运行时，JobTracker既负责资源管理又负责任务调度,集群繁忙时JobTracker容易成为瓶颈
* 可用性差：单节点Master 没有备用Master及选举操作
* 资源利用率低：TaskTracker使用slot等量划分本节点上的资源量。slot代表(CPU、内存等)一个Task获取到一个slot后才有机会运行 一些Task不能充分利用slot，其他Task也无法使用空闲资源 slot分为Map slot 和Reduce Task
* 不能支持多种MapReduce框架：无法通过可插拔方式将自身的MapReduce框架替换为其他实现 Spark、Storm等

MRv2重用了MRv1中的编程模型和数据处理引擎，运行时环境被重构

JobTracker拆分成通用的资源调度平台(ResourceManager RM)和负责各个计算框架的任务调度模型(ApplicationMaster AM)

MRv2中MapReduce核心不再是MapReduce框架 而是YARN MapReduce可插拔

MRv2虽然解决了MRv1中的一些问题，但由于对HDFS频繁操作(计算结果持久化、数据备份、shuffle等) 导致磁盘I/O成为系统性能瓶颈，不能提供实时能力
### Spark使用场景
Hadoop常用于解决高吞吐、批量统计 Spark通过内存计算能力极大的提高了大数据处理速度
### Spark的特点
对MapReduce大量优化

* 快速处理能力。MapReduce的Job将中间输出和结果存储在HDFS中，读写HDFS造成磁盘I/O成瓶颈。Spark允许将中间输出和结果存储在内存中 Spark自身的DAG执行引擎也支持数据在内存中的计算
* 易于使用。支持各种语言
* 支持查询。Spark支持SQL及Hive SQL
* 支持流式计算。Spark Streaming对数据实时处理
* 可用性高。Spark Standalone部署模式 Master可以有多个，解决了单点故障问题，可以使用YARN、Mesos等替换
* 丰富的数据源支持。除了HDFS 还可访问Cassandra HBase Hive Tachyon以及任何Hadoop数据源
### Spark基本概念
* RDD(resillient distributed dataset):弹性分布式数据集
* Task:具体执行任务。Task分为ShuffleMapTask和ResultTask两种 ShuffleMapTask和ResultTask分别类似与Hadoop中的Map和Reduce
* Job:用户提交的作业。一个作业可能由一到多个Task组成。
* Stage:Job分成的阶段。一个Job可能被划分成一到多个Stage。
* Partition:数据分区。即一个RDD的数据可以划分为多个分区
* NarrowDependency:窄依赖，即子RDD依赖于父RDD中固定的Partition NarrowDependency分为OneToOneDependency和RangeDependency两种
* ShuffleDependcy:shuffle依赖，也成为宽依赖。即子RDD对父RDD中的所有Partition都有依赖。
* DAG(directed acycle graph):有向无环图。用于反映各RDD之间的依赖关系。
### Spark模块设计
* Spark Core:Spark的核心功能实现，包括：SparkContext的初始化(Driver Application 通过SparkContext提交)、部署模式、存储体系、任务提交与执行、计算引擎等。
* Spark SQL:提供SQL处理能力，便于熟悉关系数据库操作的工程师进行交互查询。还为熟悉Hadoop用户提供Hive SQL处理能力
* Spark Streaming：提供流式计算处理能力，目前支持Kafka、Flume、ZeroMQ等数据源。此外，还提供窗口操作。
* GraphX:提供图计算能力。支持分布式
* MLlib:提供机器学习相关的统计、分类、回归等领域的多种算法实现。

### Spark Core
* SparkContext: Driver Application的执行与输出都是通过SparkContext来完成的 正式提交Application之前，需要初始化SparkContext
                SparkContext隐藏了网络通信、分布式部署、消息通信、存储能力、计算能力、缓存、测量系统、文件服务、Web服务等内容。开发者通过API完成功能开发
                SparkContext内置的DAGScheduler负责创建Job，将DAG中的RDD划分到不同的Stage 提交Stage等功能
                内置的TaskScheduler负责资源的申请、任务的提交及请求集群对任务的调度等工作
* 存储体系：Spark优先考虑各节点的内存作为存储，当内存不足时会考虑使用磁盘，极大的减少了磁盘I/O 使得Spark适用于实时计算。
          Spark还提供了以内存为中心的高容错分布式文件系统Tachyon供用户进行选择。Tachyon能够为Spark提供可靠的内存级的文件共享服务
* 计算引擎：由SparkContext中DAGScheduler、RDD以及具体节点上的Executer负责执行计算的Map和Reduce任务组成。          
          DAGScheduler和RDD虽然位于SparkContext内部，但是在任务正式提交与执行之前会将Job中的RDD组织成有向无关图(DAG) 并对Stage进行划分，决定了任务执行阶段任务的数量、迭代计算、shuffle等过程
* 部署模式：由于单节点不足以提供足够的存储及计算能力，所以Spark在SparkContext的TaskScheduler组件中提供了对Standalone部署模式的实现和Yarn、Mesos等分布式资源管理系统的支持
          通过使用Standalone、Yarn、Mesos等部署模式为Task分配计算资源，提高任务的并发执行效率
### Spark 扩展功能
* Spark SQL:首先使用SQL语句解析器(SplParser)将SQL转换为语法树(Tree),并且使用规则执行器(RuleExecutor)将一系列规则(Rule)应用到语法树，最终生成物理执行计划并执行 。其中规则执行器包括语法分析器(Analyzer)和优化器(Optimizer)
* Spark Streaming：输入流接收器(Receiver)负责接入数据，是接入数据流的接口规范。Dstream是Spark Streaming中所有数据流的抽象。Dstream本质上由一系列连续的RDD组成
* GraphX
* MLlib

### Spark模型设计
#### Spark编程模型
1. 用户使用SparkContext提供的API(常用的有textFile、sequenceFile、runJob、stop等)编写Driver application程序。此外SQLContext、HiveContext及StreamingContext对SparkContext进行封装，并提供了SQL、Hive及流式计算相关API
2. 使用SparkContext提交的用户应用程序，首先会使用BlockManager和BroadcastManager将任务的Hadoop配置进行广播。然后由DAGScheduler将任务转换为RDD并组织成DAG，DAG还将被划分为不同的Stage。最后由TaskScheduler借助ActorSystem将任务提交给集群管理器(Cluster Manager)
3. 集群管理器(Cluster Manager)给任务分配资源，即将具体任务分配到Worker上，Worker创建Executor来处理任务的运行。
#### RDD计算模型
RDD可以看做是对各种计算模型的统一抽象，Spark的计算过程主要是RDD的迭代计算过程，分区数量取决于partition数量的设定，每个分区的数据只会在一个Task中计算。所有分区可以在多个机器节点的Executor上并行执行。

### Spark基本架构
从集群部署角度，Spark集群由以下部分组成:
* Cluster Manager：Spark集群管理器，主要负责资源的分配与管理。集群管理器分配的资源属于一级分配，它将各个Worker上的内存、CPU等资源分配给应用程序，但不负责对Executor的资源分配
* Worker:Spark的工作节点。 对Spark应用程序来说，由集群管理器分配得到资源的Worker节点主要负责以下工作。创建Executor,将资源和任务进一步分配给Executor 同步资源信息给Cluster Manager
* Executor：执行计算任务的一线进程。主要负责任务的执行以及与Worker、Driver App的信息同步          
* Driver App:客户端驱动程序，也可以理解为客户端应用程序，用于将任务程序转换为RDD和DAG，并与Cluster Manager进行通信与调度
## SparkContext的初始化
### SparkContext概述
Spark Driver用于提交用户应用程序，可以看做Spark客户端，SparkContext初始化完毕，才能向Spark集群提交任务。 SparkContext的配置参数则有SparkConf负责

SparkConf主要通过ConcurrentHashMap来维护各种以"spark."开头的字符串

初始化步骤:
1. 创建Spark执行环境SparkEnv
2. 创建RDD清理器metadataCleaner
3. 创建并初始化Spark UI
4. Hadoop 相关配置及Executor环境变量的设置
5. 创建任务调度TaskScheduler
6. 创建和启动DAGScheduler
7. TaskScheduler的启动
8. 初始化块管理器BlockManager(BlockManager是存储体系的主要组件之一)
9. 启动测量系统MetricsSystem
10. 创建和启动Executor分配管理器ExecutorAllocationManager
11. ContextCleaner的创建与启动
12. Spark环境更新
13. 创建DAGSchedulerSource和BlockManagerSource
14. 将SparkContext标记为激活

### 创建执行环境SparkEnv
SparkEnv是Spark的执行环境对象，其中包括众多与Executor执行相关的对象。由于在local模式下Driver会创建Executor，local-cluster部署模式或者Standalone部署模式下Worker另起的CoarseGrainedExecutorBackend进程中也会创建Executor,
所以SparkEnv存在于Driver或者CoarseGrainedExecutorBackend进程中。创建SparkEnv主要使用SparkEnv的createDriverEnv 有三个参数:conf、isLocal和listenerBus

conf为SparkConf的复制 isLocal标识是否是单击模式 listenerBus采用监听器模式维护各类事件的处理

SparkEnv的方法createDriverEnv最终调用create创建SparkEnv 构造步骤如下：
1. 创建安全管理器SecurityManager
2. 创建基于Akka的分布式消息系统ActorSystem
3. 创建Map任务输出跟踪器mapOutputTracker
4. 实例化ShuffleManager
5. 创建ShuffleMemoryManager
6. 创建块传输服务BlockTransferService
7. 创建BlockManagerMaster
8. 创建块管理器BlockManager
9. 创建广播管理器BroadcastManager
10. 创建缓存管理器CacheManager
11. 创建HTTP文件服务器HttpFileServer
12. 创建SparkEnv
#### 安全管理器SecurityManager
主要对权限、帐号进行设置，如果使用Hadop YARN作为集群管理器则需要使用证书生成secret key登录 最后给当前系统设置默认的口令认证实例，采用匿名内部类实现
#### 基于Akka的分布式消息系统ActorSystem
ActorSystem是Spark中最基础的设施，Spark既使用它发送分布式消息，又用它实现并发编程。

消息系统可实现并发？Scala认为Java线程通过共享数据以及通过锁来维护共享数据的一致性是糟糕的做法，容易引起锁的争用 降低并发程序的性能甚至死锁

在scala中只需要自定义类型继承Actor，并且提供act方法，就如同Java实现Runnable接口，需要实现run方法一样。但不能直接调用act方法，而是通过发送消息的方式(Scala发送消息是异步的)传递数据
```
Actor ! message
```
Akka是Actor编程模型的高级类库，ActorSystem是Akka提供的用于创建分布式消息通信系统的基础类

Spark的Driver中Akka的默认访问地址是akka://sparkDriver Executor中Akka默认的访问地址是akka://SparkExecutor 如果不指定ActorSystem的端口 那么所有节点ActorSystem端口每次启动时随机产生
#### map任务输出跟踪器mapOutputTracker
用于跟踪map阶段任务的输出状态，此状态便于reduce阶段任务获取地址及中间输出结果 每个map任务或者reduce任务都会有其唯一标识 mapId和reduceId

每个reduce任务的输入可能是多个map任务的输出 reduce会到各个map所在的节点拉去Block，这一过程叫shuffle 每批shuffle过程都有唯一的标识shuffleId

mapOutputTrackerMaster：内部使用mapStatuses:TimestampHashMap[Int,Array[MapStatus]]来维护跟踪各个map任务的输出状态 key对应shuffleId，Array存储各个map任务对应的状态信息 MapStatus

MapStatus维护了map输出Block的地址BlockManagerId(所以reduce任务知道从何处获取map任务的中间输出)

mapOutputTrackerMaster还用cachedSerializedStatuses:TimestampHashMap[Int,Array[Byte]]维护序列化后的各个map任务的输出状态。key对应shuffleId Array存储各个序列化MapStatus生成的字节数组

Dirver和Executor处理mapOutputTrackerMaster的方式不同：
* Driver 则创建mapOutputTrackerMaster，然后创建mapOutputTrackerMasterActor 并且注册到ActorSystem中
* Executor 则创建mapOutputTrackerWorker,并从ActorSystem中找到mapOutputTrackerMasterActor
无论Driver还是Executor 最后都由mapOutputTracker的属性trackerActor持有mapOutputTrackerMasterActor的引用

map任务的状态正是由Executor向持有的mapOutputTrackerMasterActor发送消息。将map任务状态同步到mapOutputTracker的mapStatuses和cachedSerializedStatuses的
#### 实例化ShuffleManager
ShuffleManager负责管理本地及远程的block数据的shuffle操作。默认为通过反射方式生成的SortShuffleManager的实例 可以修改属性spark.shuffle.manager为hash来显式控制使用HashShuffleManager

SortShuffleManager通过持有的IndexShuffleBlockManager间接操作BlockManager中的DiskBlockManager将map结果写入本地 根据shuffleId、mapId写入索引文件，也能通过mapOutputTrackerMaster中的mapStatuses从本地或者其他远程节点读取文件

为什么需要shuffle？

Spark作为并行计算框架，同一个作业会被划分为多个任务在多个节点并行执行 reduce的输入可能存在于多个节点上，因此需要通过洗牌将所有reduce的输入汇总起来，这个过程就是shuffle

#### shuffle线程内存管理器ShuffleMemoryManager
ShuffleMemoryManager负责管理shuffle线程占有内存的分配和释放，并通过threadMemory:mutable.HashMap[Long,Long]缓存每个线程的内存字节数

getMaxMemory方法用于获取shuffle所有线程占用的最大内存 计算公式：Java运行时最大内存*Spark的shuffle最大内存占比*Spark的安全内存占比

ShuffleMemoryManager通常运行在Executor中，Driver中的ShuffleMemoryManager只有在local模式下才起作用
#### 块传输服务BlockTransferService
BlockTransferService默认为NettyBlockTransferService(可以配置属性spark.shuffle.BlockTransferService使用NioBlockTransferService)它使用Netty提供的异步事件驱动的网络应用框架，提供Web服务及客户端，获取远程节点上Block的集合
#### BlockManagerMaster介绍
BlockManagerMaster负责对Block的管理和协调，具体操作依赖于BlockManagerMasterActor。Driver和Executor处理BlockManagerMaster的方式不同
* 如果当前应用程序是Driver，则创建BlockManagerMasterActor，并注册到ActorSystem
* 如果当前应用程序是Executor，则从ActorSystem中找到BlockManagerMasterActor。

无论Driver或是Executor 最后BlockManagerMaster的属性driverActor将持有对BlockManagerMasterActor的引用
#### 创建块管理器BlockManager
BlockManager负责对Block的管理，只有在BlockManager的初始化方法initialize被调用后，它才是有效的。BlockManager作为存储系统的一部分
#### 创建广播管理器BroadcastManager
BroadcastManager用于将配置信息和序列化后的RDD、Job以及ShuffleDependency等信息在本地存储。如果为了容灾，也会复制到其他节点上

BroadcastManager必须在其初始化方法initialize被调用后，才能生效。initialize方法实际利用反射生成广播工厂实例broadcastFactory(可配置属性spark.broadast.factory指定，默认org.apache.spark.broadcast.TorrentBroadcastFactory)

BroadcastManager的广播方法newBroadcast实际代理了工厂broadcastFactory的newBroadcast方法来生成广播对象。 unbroadcast方法实际代理了工厂broadcastFactory的unbroadcast方法生成非广播对象
#### 创建缓存管理器CacheManager
CacheManager用于缓存RDD某个分区计算后的中间结果，缓存计算结果发生在迭代计算的时候
#### HTTP文件服务器HttpFileServer
HttpFileServer主要提供对jar及其他文件的http访问，这些jar包包括用户上传的jar包 端口由spark.fileserver.port 默认为0表示随机生成端口号

HttpFileServer初始化过程：
1. 使用Utils工具类创建文件服务器的根目录及临时目录(临时目录在运行时环境关闭时删除)
2. 创建存放jar包及其他文件的文件目录
3. 创建并启动HTTP服务
#### 创建测量系统MetricsSystem
MetricsSystem是Spark的测量系统
#### 创建SparkEnv
所有基础组件准备好后，最终创建执行环境SparkEnv

serializer和closureSerializer都是使用Class.forName反射生成的org.apache.apark.serializer.JavaSerializer类的实例，其中closureSerializer实例特别用来对Scala中的闭包进行序列化




### 创建metadataCleaner
SparkContext为了保持对所有持久化的RDD的跟踪，使用类型是TimeStampedWeakValueHashMap的persistentRdds缓存。metadataCleaner的功能是清除过期的持久化RDD

实质是一个用于TimeTask实现的定时器 不断调用cleanupFunc 构造metadataCleaner时的函数是cleanup，用于清理persistentRdds中的过期内容
### SparkUI
大型分布式系统中，采用事件监听机制是最常见的 假如SparkUI采用函数调用方式，随着集群规模增加，函数调用越来越多，最终会受到Driver所在JVM的线程数量限制而影响监控数据的更新，甚至无法即使显示给用户的情况，因为函数调用多数是同步调用

分布式环境还可能因为网络问题导致线程长时间占用，将函数调用更换为发送事件，事件处理是异步的 整个系统的并发度会大大增加。发送的事件会存入缓存，由定时调度器取出后，分配给监听此事件的监听器对监控数据进行更新

* DAGScheduler是主要的产生各类SparkListenerEvent的源头，它将各种SparkListenerEvent发送到listenerBus的事件队列中
* listenerBus通过定时器将SparkListenerEvent事件匹配到具体的SparkListener,改变SparkListener中的统计监控数据，最终由SparkUI的界面展示

#### listenerBus
listenerBus的类型是LiveListenerBus LiveListenerBus实现了监听器模型，通过监听事件触发对各种监听器监听状态信息的修改，达到UI界面刷新效果，LiveListenerBus由以下部分组成：
* 事件阻塞队列：类型为LinkedBlockingQueue[SparkListenerEvent] 固定大小10000
* 监听器数组：类型为ArrayBuffer[SparkListener],存放各类监听器SparkListener
* 事件匹配监听器的线程：此Thread不断拉取LinkedBlockingQueue中的事件，遍历监听器，调用监听器的方法。任何事件都会在LinkedBlockingQueue中存在一段时间，然后Thread处理了此事件后，会将其删除
#### 构造JobProgressListener
以JobProgressListener为例来了解SparkListener JobProgressListener是SparkContext中一个重要组成部分，通过监听listenerBus中的事件更新任务进度。

SparkStatusTracker和SparkUI实际上也是通过JobProgressListener来实现任务状态跟踪

JobProgressListener的作用是通过HashMap、ListBuffer等数据结构存储JobId及对应的JobUIData信息 并按照激活、完成、失败等job状态统计 对于StageId、StageInfo等信息按照激活、完成、忽略、失败等Stage状态统计，并且存储Stage与JobId的一对多关系

这些统计信息最终会被JobPage和StagePage等页面访问和渲染。
#### SparkUI的创建与初始化
create方法除了JobProgressListener是外部传入的之外又增加了一些SparkListener
* 用于维护Executor的存储状态的StorageStatusListener
* 用于准备将Executor的信息展示在ExecutorsTab的ExecutorsListener
* 用于准备将Executor相关存储信息展示在BlockManagerUI的StorageStatusListener

最后创建SparkUI,SparkUI服务默认是可以被杀掉的，通过修改soark.ui.killEnableed为false可以保证不被杀死

#### Spark UI的页面布局与展示
#### Spark UI的启动
### Hadoop相关配置及Executor环境变量
#### Hadoop相关配置信息
默认情况下，Spark使用HDFS作为分布式文件系统，获取的配置信息包括：
* 将Amazon S3文件系统的AccessKeyId和SecretAccessKey加载到Hadoop的configuration
* 将SparkConf中所有以spark.hadoop.开头的属性都复制到Hadoop的Configuration
* 将SparkConf的属性spark.buffer.size复制为Hadoop的Configuration的配置io.file.buffer.size

如果指定了SPARK_YARN_MODE属性，则会使用YarnSparkHadoopUtil否则为SparkHadoopUtil
#### Executor环境变量
对Executor的环境变量的处理，Master给Worker发送调度后，Worker最终使用executorEnvs提供的信息启动Executor 可以通过配置spark.executor.memory指定Executor占用的内存大小 也可以配置系统变量SPARK_EXECUTOR_MEMORY或者SPARK_MEM对其大小进行设置
### 创建任务调度器TaskScheduler
TaskScheduler也是SparkContext的重要组成部分，负责任务的提交，并且请求集群管理器对任务调度。TaskScheduler也可以看做任务调度的客户端
