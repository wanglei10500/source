---
title: hadoop
tags:
 - hadoop
categories: 经验分享
---
### 部署模式
Spark支持的部署模式：
* 本地部署模式：local、local[N]、local[N,maxRetries]。主要用于代码调试和跟踪，不具备容错能力，所以不适用生产环境
* 本地集群部署模式：local-cluster[N,cores,memory]。也主要用于代码调试和测试，是源码学习常用的模式。不具备容错能力，不能用于生产环境。
* Standalone部署模式：spark://。具备容错能力并且支持分布式部署，所以可用于实际的生产。
* 第三方部署模式：yarn-standalone、yarn-cluster、mesos://、zk://、sime://等。
概念：
* Driver:应用驱动程序，理解为老板的客户
* Master:Spark的主控节点，可以理解为集群的老板
* Worker:Spark的工作节点，可以理解为集群的各个主管
* Executor:Spark的工作进程，由Worker监管，负责具体任务的执行
### local部署模式
local部署模式只有Driver，没有Master和Worker，执行任务的Executor和Driver在同一个JVM进程内。

local模式下的任务提交与执行过程：
1. local模式下ExecutorBackend的实现类是LocalBackend，当有任务要提交时，由TaskSchedulerImpl调用LocalBackend的reviveOffers方法申请资源
2. LocalBackend向LocalActor发送reviveOffers消息，为任务申请资源
3. LocalActor收到ReviveOffers消息后，调用TaskSchedulerImpl的resourceOffers方法申请资源，TaskSchedulerImpl将根据任务申请的CPU核数和内存及本地化等条件为其分配资源
4. 任务获得资源后，调用Executor的launchTask方法运行任务
5. 在任务运行过程中，Executor中运行的TaskRunner通过调用LocalBackend的statusUpdate方法，实现向LocalActor发送statusUpdate消息。LocalActor接收到statusUpdate消息，再调用TaskSchedulerImpl的statusUpdate不断更新任务的状态。任务的状态有LAUNCHING、RUNNING、FINISHED、FAILED、KILLED和LOST等。
### local-cluster部署模式
local-cluster是一种伪集群部署模式，Driver、Master和Worker在同一个JVM进程内，可以存在多个Worker，每个Worker会有多个Executor，但这些Executor都独立存在于一个JVM进程内，与local部署模式区别？
* 使用LocalSparkCluster启动集群
* SparkDeploySchedulerBackend的启动过程
* AppClient的启动与调度
* local-cluster模式的任务执行

若设置master为 local-cluster[2,1,1024]
* 2指定了Worker的数量(numSlaves)
* 1指定了每个Worker占用的CPU核数(coresPerSlave)
* 1024指定的是每个Worker占用的内存大小(memoryPerSlave) memoryPerSlave必须比executorMemory大，因为Worker的内存大小包括Executor占用的内存

与local模式不同的是，local-cluster除TaskSchedulerImpl外，还创建了LocalSparkCluster，LocalSparkCluster的start方法用于启动集群。local-cluster模式中使用的ExecutorBackend的实现类是SparkDeploySchedulerBackend

给SparkDeploySchedulerBackend的shutdownCallback绑定LocalSparkCluster的stop方法，用于当Driver关闭时关闭集群，仅限于local-cluster模式
#### LocalSparkCluster的启动
* mastertActorSystems:用于缓存所有的Master的ActorSystem
* workerActorSystems:维护所有的Worker的ActorSystem

LocalSparkCluster的start方法用于创建、启动Master的ActorSystem与多个Worker的ActorSystem， LocalSparkCluster的stop方法关闭、清理Master的ActorSystem与多个Worker的ActorSystem

 略
### Standalone部署模式
local模式只有Driver和Executor，且在同一个JVM进程内。local-cluster模式的Driver、Master、Worker也都位于同一个JVM进程内部。所以local、local-cluster便于开发、测试、源码阅读与调试

Standalone哪些特点
* Driver在集群之外，可以是任意的客户端应用程序
* Master部署于单独的进程，甚至应该在单独的机器节点上。Master有多个，但同时最多只有一个处于激活状态
* Worker部署于单独的进程，也推荐在单独的机器节点上部署
#### 启动Standalone模式
以linux环境下启动Standalone模式为例。 顺序：先启动Master再逐个启动Worker。

启动Master命令/spark-bin-hadoop1/sbin|$ sh start-master.sh 默认端口7077 webui端口8080

启动Worker Worker创建并启动了自己的ActorSystem:akka.tcp://sparkWorker@localhost:53728 WebUI端口8081,最后向Master注册Worker等信息。  $SPARK_HOME/sbin/start-slaves.sh来启动多个Worker

#### 启动Master分析
main函数完成的内容如下：
1. 创建SparkConf。
2. Master参数解析。
3. 创建、启动ActorSystem ,并向ActorSystem注册  Master
#### 启动Worker分析
main函数完成的内容如下：
1. 创建SparkConf
2. Worker参数解析
3. 创建、启动ActorSystem，并向ActorSystem注册Worker
#### 启动Driver Application分析
Standalone与local-cluster模式启动区别：
1. 集群是真正的分布式部署，所有Master和Worker都位于独立的JVM进程甚至是不同的机器节点上
2. Standalone模式下可以存在多个Master，这些Master之间通过持久化引擎和领导选举机制解决生成环境下Master的单点问题，使得Master在异常退出后，能够重新选举激活状态的Master，并从故障中恢复集群
#### 资源回收
Application占用的资源应该被回收，那么Master和Executor是如何感知到Application的退出的？
Spark中有两种处理方式：一种是离别之前先打声招呼，一种是不辞而别
##### 离别之前先打声招呼
SparkContext提供了stop方法
##### 不辞而别
上面Application只跟Executor打声招呼，却忘记了Master

Akka的通信机制保证当相互通信的任意一方异常退出，另一方都会收到DisassociatedEvent。Master正是在处理DisassociatedEvent消息时移除已经停止的Driver Application

如果忘记调用stop方法，Executor的资源虽然不会被主动回收，但由于CoarseGrainedExecutorBackend也会收到DisassociatedEvent的消息，直接退出CoarseGrainedExecutorBackend进程
### 容错机制
在分布式系统中，由于机器数量众多，所以机器发生故障的概率很高，在设计任何分布式系统都要考虑容错，本节针对Standalone部署模式的容错能力进行分析
#### Executor异常退出
如果Executor异常退出，那么势必导致在此Executor上执行的任务无法运行 一旦收到退出状态(EXITED),则会向Worker发送ExecutorStateChanged消息。Worker收到ExecutorStateChanged消息后向Master转发ExecutorStateChanged

Master收到ExecutorStateChanged消息后对退出状态的处理步骤如下：
1. 找到占有Executor的Application的ApplicationInfo，以及Executor对应的ExecutorInfo
2. 将ExecutorInfo的状态改为EXITED
3. EXITED也属于Executor完成状态，所以会将ExecutorInfo从ApplicationInfo和WorkerInfo中移除。
4. 由于Executor是非正常退出，所以重新调用scheduler给Application给Application进行资源调度
#### Master异常退出
如果Standalone部署集群只有一个Master，当Master退出时发生什么？
1. 当Executor上的任务执行完毕，需要对资源回收。 如果主动调用SparkContext的top方法，Driver Application最终会向给自己服务的Executor发送StopExecutor消息对资源回收
Master虽然退出 但是Driver Application依然持有这些Executor，所以没有影响。 如果不辞而别，那么Master将收不到DisassociatedEvent消息，但是CoarseGrainedExecutorBackend仍然可以收到DisassociatedEvent消息后退出进程。
看来Master退出，如果Worker、Executor正常运行，则对于资源回收没有影响
2. 此时如果再发生Executor异常退出问题，Worker无法通过ExecutorStateChanged消息促使Master重新给Driver Application调度运行Executor，Driver Application只能眼巴巴看着自己提交的任务无法执行。而Executor占用的资源也得不到回收
3. 此时如果又发生Worker异常退出问题，那么Worker和Executor都将停止服务，由于无法通知Master重新给Driver Application调度到其他Worker上 Driver Application提交的任务也将无法执行。Worker虽然kill掉了Executor，但是Worker的资源将无法被其他Driver Application使用
4. 有新的Driver需要提交任务则无法成功
解决此问题的常用方式就是采用Master/Slaves架构，即有多个Master，但同时只有一个Master负责整个集群的调度、资源管理工作

两个与Master/Slaves架构有关的组件：故障恢复的持久化引擎(persistenceEngine)和领导选举代理(leaderElectionAgent)

故障恢复的持久化引擎有如下几种：
* ZooKeeperPersistenceEngine:基于ZooKeeper实现的持久化引擎
* FileSystemPersistenceEngine:基于文件系统的持久化引擎
* BlackHolePersistenceEngine:默认的持久化引擎，实际上并不提供故障恢复的持久化能力

领导选举机制(Leader Election)可以保证集群虽然存在多个Master，但是只有一个Master处于激活(Active)状态，其他的Master(Standby)状态。当Active状态的Master出现故障，会选举出一个Standby状态的Master作为新的Active状态的Master。
由于整个集群的Worker、Driver和Application的信息都已经持久化到文件系统，因此切换时只会影响新任务的提交，对于正在运行中的任务没有任何影响。选举机制的代理有两种：
* ZooKeeperLeaderElectionAgent：对ZooKeeper提供的选举机制的代理。
* MonarchyLeaderAgent:默认的选举机制代理
默认情况下，Spark不提供故障恢复

当没有设置spark.deploy.recoveryDirectory和spark.deploy.recoveryMode时，RECOVERY_MODE等于NONE,此时匹配的持久化引擎是BlackHolePersistenceEngine.BlackHolePersistenceEngine虽然继承了persistenceEngine，但实现的是空方法

如果不设置spark.deploy.recoveryDirectory和spark.deploy.recoveryMode，选择切换之后新的Master会丢失集群之前的所有信息
1. FileSystemPersistenceEngine搭配MonarchyLeaderAgent实现故障恢复
2. 使用ZooKeeper提供的选举和持久化
### 其他部署方案
#### YARN
MRv1的运行时环境由JobTracker和TaskTracker两类服务组成，JobTracker负责资源和任务的管理与调度，TaskTracker负责单个节点的资源管理和任务执行。

在YARN中，JobTracker被分为两部分：ResourceManager(RM) 和ApplicationMaster(AM) RM负责资源管理和调度，AM负责具体应用程序的任务划分，调度等工作。

YARN的基本结构包括以下部分：
* ResourceManager(RM):全局资源管理器，负责整个系统的资源管理与分配。RM由调度器和应用程序管理器组成。调度器将系统资源分配给各个应用程序，资源的分配单位不再是MRv1中的“slot”，而是Container。
  Container是对CPU、内存等资源的封装。应用程序管理器负责管理整个系统的应用程序，如处理程序提交，与调度器沟通后为应用程序启动ApplicationMaster等。
* ApplicationMaster(AM):用户提交的每个任务都有一个AM，他会与RM通信获取资源，将任务划分为更细粒度的任务，与NodeManager(NM)通信启动或停止任务，监控失败任务为其重新申请资源后重新启动等
* NodeManager(NM): 单个节点上的资源与任务管理器，它负责向RM定时汇报本节点的资源使用情况及各个Container的状态，也接受处理AM的启动或停止任务请求。

YARN对于支持的MapReduce框架是可插拔的，Spark对于集群管理器也支持可插拔，两者不谋而合。

与YARN集成时整个Spark集群的启动顺序如下。
1. 将Spark提供的ApplicationMaster在YARN集群中启动。
2. ApplicationMaster向ResourceManager申请Container
3. 申请Container成功后，向具体的NodeManager发送指令启动Container
4. ApplicationMaster启动对各个运行的ContainerExecutor进行监控

设置masteer为”yarn-cluster“，那么在创建TaskSchedulerImpl时就会匹配yarn-cluster模式， 其中yarnClusterScheduler继承自TaskSchedulerImpl，因此yarnClusterScheduler将负责任务的提交与调度。

YarnSchedulerBackend继承自org.apache.spark.scheduler.cluster包中的CoarseGrainedExecutorBackend，yarnClusterSchedulerBackend又继承了YarnSchedulerBackend 于是yarnClusterSchedulerBackend就是TaskSchedulerImpl的backend

Driver Application初始化完毕会向ApplicationMaster进行注册，在YARN部署模式中，Worker已被NodeManager替代，ApplicationMaster给Application分配资源主要借助YarnAllocationHandle
#### mesos
### 小结
