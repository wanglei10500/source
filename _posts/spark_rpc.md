---
title: spark RPC
tags:
 - spark
categories: 经验分享
---

### Spark RPC通信机制

Spark RPC中最为重要的三个抽象：RpcEnv、RpcEndpoint、RpcEndpointRef，这样做的好处有：
* 对上层API来说，屏蔽了底层的具体实现，使用方便
* 可以通过不同的实现来完成指定的功能，方便扩展
* 促进了底层实现层的良性竞争，Spark 1.6.3中默认使用了Netty作为底层的实现，但Akka的依赖依然存在；而Spark 2.1.0中的底层实现只有Netty，这样用户可以方便的使用不同版本的Akka或者将来某种更好的底层实现
#### send a message locally
 Rpc发送本地消息测试代码 test org.apache.spark.rpc.RpcEnvSuite
```
  test("send a message locally") {
    @volatile var message: String = null
    val rpcEndpointRef = env.setupEndpoint("send-locally", new RpcEndpoint {
      override val rpcEnv = env

      override def receive = {
        case msg: String => message = msg
      }
    })
    rpcEndpointRef.send("hello")
    eventually(timeout(5 seconds), interval(10 millis)) {
      assert("hello" === message)
    }
  }
```
![spark](http://upload-images.jianshu.io/upload_images/4473093-41a79db809742cb3.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/690)
1. 创建RpcEndpoint，并初始化rpcEnv的引用（RpcEnv已经创建好，底层实际上是实例化了一个NettyRpcEnv，而NettyRpcEnv是通过工厂方法NettyRpcEnvFactory创建的）
2. 实例化RpcEndpoint之后需要向RpcEnv注册该RpcEndpoint，底层实现是向NettyRpcEnv进行注册，而实际上是通过调用Dispatcher的registerRpcEndpoint方法向Dispatcher进行注册
3. 具体的注册就是向endpoints、endpointRefs、receivers中插入记录：而receivers中插入的信息会被Dispatcher中的线程池中的线程执行：会将记录take出来然后调用Inbox的process方法通过模式匹配的方法进行处理，注册的时候通过匹配到OnStart类型的message，去执行RpcEndpoint   的onStart方法（例如Master、Worker注册时，就要执行各自的onStart方法），本例中未做任何操作
4. 注册完成后返回RpcEndpointRef，我们通过RpcEndpointRef就可以向其代表的RpcEndpoint发送消息

通过RpcEndpointRef向其代表的RpcEndpoint发送消息的具体流程：（图中的红色线条部分）
1. (1 2) 调用RpcEndpointRef的send方法，底层实现是调用Netty的NettyRpcEndpointRef的send方法，而实际上又是调用的NettyRpcEnv的send方法，发送的消息使用RequestMessage进行封装：
```
nettyEnv.send(RequestMessage(nettyEnv.address, this, message))
```
3. (3 4) NettyRpcEnv的send方法首先会根据RpcAddress判断是本地还是远程调用，此处是同一个RpcEnv，所以是本地调用，即调用Dispatcher的postOneWayMessage方法
5. postOneWayMessage方法内部调用Dispatcher的postMessage方法
6. postMessage会向具体的RpcEndpoint发送消息，首先通过endpointName从endpoints中获得注册时的EndpointData，如果不为空就执行EndpointData中Inbox的post(message)方法，向Inbox的mesages中插入一条InboxMessage，同时向receivers中插入一条记录，此处将Inbox单独画出来是为了方便大家理解
7. Dispatcher中的线程池会拿出一条线程用来循环receivers中的消息，首先使用take方法获得receivers中的一条记录，然后调用Inbox的process方法来执行这条记录，而process将messages中的一条InboxMessage（第6步中插入的）拿出来进行处理，具体的处理方法就是通过模式匹配的方法，匹配到消息的类型（此处是OneWayMessage），然后来执行RpcEndpoint中对应的receive方法，在此例中我们只打印出这条消息（步骤8）
![spark](http://upload-images.jianshu.io/upload_images/4473093-8cb4e4eccbc61bba.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/500)
通过NettyRpcEndpointRef来发出一个消息，消息经过NettyRpcEnv、Dispatcher、Inbox的共同处理最终将消息发送到NettyRpcEndpoint，NettyRpcEndpoint收到消息后进行处理（一般是通过模式匹配的方式进行不同的处理）
![spark](http://upload-images.jianshu.io/upload_images/4473093-e2dc3415aff1c163.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/500)
RpcEndpointRef发送消息给RpcEnv，RpcEnv查询注册信息将消息路由到指定的RpcEndpoint，RpcEndpoint接收到消息后进行处理（模式匹配的方式）

RpcEndpoint的声明周期：constructor -> onStart -> receive* -> onStop

其中receive*包括receive和receiveAndReply

park Rpc就是Spark中对分布式消息通信系统的高度抽象。
![spark](https://wongxingjun.github.io/figures/spark-rpc/spark-rpc-rpcenv.png)
RPC默认采用Netty实现。RpcEndpoint注册到RpcEnv上然后才能接收消息，RpcEnv处理从RpcEndpointRef或者其他远程节点的消息，然后回发给相应的RpcEndpoint。
### master实现
#### master的定义
org.apache.spark.deploy.master.Master
```
private[deploy] class Master(
    override val rpcEnv: RpcEnv,
    address: RpcAddress,
    webUiPort: Int,
    val securityMgr: SecurityManager,
    val conf: SparkConf)
  extends ThreadSafeRpcEndpoint with Logging with LeaderElectable
```
master类继承自ThreadSafeRpcEndpoint，进一步定位可以发现ThreadSafeRpcEndpoint是继承自RpcEndpoint特质。
#### master启动
org.apache.spark.deploy.master.Master
```
def startRpcEnvAndEndpoint(
    host: String,
    port: Int,
    webUiPort: Int,
    conf: SparkConf): (RpcEnv, Int, Option[Int]) = {
  val securityMgr = new SecurityManager(conf)
  val rpcEnv = RpcEnv.create(SYSTEM_NAME, host, port, conf, securityMgr)
  val masterEndpoint = rpcEnv.setupEndpoint(ENDPOINT_NAME,
    new Master(rpcEnv, rpcEnv.address, webUiPort, securityMgr, conf))
  val portsResponse = masterEndpoint.askSync[BoundPortsResponse](BoundPortsRequest)
  (rpcEnv, portsResponse.webUIPort, portsResponse.restPort)
}
```
#### RpcEndpoint特质
org.apache.spark.rpc.RpcEndpoint

master的启动会创建一个RpcEnv并将自己注册到其中。继续看RpcEndpoint特质的定义：
```
private[spark] trait RpcEndpoint {
  //当前RpcEndpoint注册到的RpcEnv主子，可以类比为Akka中的actorSystem
  val rpcEnv: RpcEnv
  //直接用来发送消息的RpcEndpointRef，可以类比为Akka中的actorRef
  final def self: RpcEndpointRef = {
    require(rpcEnv != null, "rpcEnv has not been initialized")
    rpcEnv.endpointRef(this)
  }
  //处理来自RpcEndpointRef.send或者RpcCallContext.reply的消息
  def receive: PartialFunction[Any, Unit] = {
    case _ => throw new SparkException(self + " does not implement 'receive'")
  }
  //处理来自RpcEndpointRef.ask的消息，会有相应的回复
  def receiveAndReply(context: RpcCallContext): PartialFunction[Any, Unit] = {
    case _ => context.sendFailure(new SparkException(self + " won't reply anything"))
  }
  //篇幅限制，其余onError，onConnected，onDisconnected，onNetworkError，
  //onStart，onStop，stop方法此处省略
}
```
#### RpcEnv抽象类
```
private[spark] abstract class RpcEnv(conf: SparkConf) {
  private[spark] val defaultLookupTimeout = RpcUtils.lookupRpcTimeout(conf)
  //返回endpointRef
  private[rpc] def endpointRef(endpoint: RpcEndpoint): RpcEndpointRef
  //返回RpcEnv监听的地址
  def address: RpcAddress
  //注册一个RpcEndpoint到RpcEnv并返回RpcEndpointRef
  def setupEndpoint(name: String, endpoint: RpcEndpoint): RpcEndpointRef
  //通过uri异步地查询RpcEndpointRef
  def asyncSetupEndpointRefByURI(uri: String): Future[RpcEndpointRef]
  //通过uri查询RpcEndpointRef，这种方式会产生阻塞
  def setupEndpointRefByURI(uri: String): RpcEndpointRef = {
    defaultLookupTimeout.awaitResult(asyncSetupEndpointRefByURI(uri))
  }
  //通过address和endpointName查询RpcEndpointRef，这种方式会产生阻塞
  def setupEndpointRef(address: RpcAddress, endpointName: String): RpcEndpointRef = {
    setupEndpointRefByURI(RpcEndpointAddress(address, endpointName).toString)
  }
  //关掉endpoint
  def stop(endpoint: RpcEndpointRef): Unit
  //关掉RpcEnv
  def shutdown(): Unit
  //等待结束
  def awaitTermination(): Unit
  //没有RpcEnv的话RpcEndpointRef是无法被反序列化的，这里是反序列化逻辑
  def deserialize[T](deserializationAction: () => T): T
  //返回文件server实例
  def fileServer: RpcEnvFileServer
  //开一个针对给定URI的channel用来下载文件
  def openChannel(uri: String): ReadableByteChannel
}

```
```
private[spark] object RpcEnv {
  def create(
      name: String,
      host: String,
      port: Int,
      conf: SparkConf,
      securityManager: SecurityManager,
      clientMode: Boolean = false): RpcEnv = {
    val config = RpcEnvConfig(conf, name, host, port, securityManager, clientMode)
    new NettyRpcEnvFactory().create(config)
  }
}

```
master启动方法中的create具体实现，可以看到调用了Netty工厂方法NettyRpcEnvFactory，该方法是对Netty的具体封装。
#### master中消息处理
在RpcEndpoint中最核心的便是receive和receiveAndReply方法，定义了消息处理的核心逻辑，master中也有相应的实现：
```
override def receive: PartialFunction[Any, Unit] = {
    case ElectedLeader =>      
    case CompleteRecovery =>
    case RevokedLeadership =>
    case RegisterApplication(description, driver) =>
    case ExecutorStateChanged(appId, execId, state, message, exitStatus) =>
    case DriverStateChanged(driverId, state, exception) =>
    case Heartbeat(workerId, worker) =>
    case MasterChangeAcknowledged(appId) =>
    case WorkerSchedulerStateResponse(workerId, executors, driverIds) =>
    case WorkerLatestState(workerId, executors, driverIds) =>
    case UnregisterApplication(applicationId) =>
    case CheckForWorkerTimeOut =>
}
```
而receiveAndReply中，定义了对需要回复的消息组的处理逻辑。
```
override def receiveAndReply(context: RpcCallContext): PartialFunction[Any, Unit] = {
    case RegisterWorker(
        id, workerHost, workerPort, workerRef, cores, memory, workerWebUiUrl) =>
    case RequestSubmitDriver(description) =>
    case RequestKillDriver(driverId) =>
    case RequestDriverStatus(driverId) =>
    case RequestMasterState =>
    case BoundPortsRequest =>
    case RequestExecutors(appId, requestedTotal) =>
    case KillExecutors(appId, executorIds) =>
}
```
### worker实现
#### worker的定义
和master同样继承自ThreadSafeRpcEndpoint特质，也是一个Endpoint类型。
```
private[deploy] class Worker(
    override val rpcEnv: RpcEnv,
    webUiPort: Int,
    cores: Int,
    memory: Int,
    masterRpcAddresses: Array[RpcAddress],
    endpointName: String,
    workDirPath: String = null,
    val conf: SparkConf,
    val securityMgr: SecurityManager)
  extends ThreadSafeRpcEndpoint with Logging {
```
#### worker启动方法
```
def startRpcEnvAndEndpoint(
    host: String,
    port: Int,
    webUiPort: Int,
    cores: Int,
    memory: Int,
    masterUrls: Array[String],
    workDir: String,
    workerNumber: Option[Int] = None,
    conf: SparkConf = new SparkConf): RpcEnv = {
  // The LocalSparkCluster runs multiple local sparkWorkerX RPC Environments
  val systemName = SYSTEM_NAME + workerNumber.map(_.toString).getOrElse("")
  val securityMgr = new SecurityManager(conf)
  val rpcEnv = RpcEnv.create(systemName, host, port, conf, securityMgr)
  val masterAddresses = masterUrls.map(RpcAddress.fromSparkURL(_))
  rpcEnv.setupEndpoint(ENDPOINT_NAME, new Worker(rpcEnv, webUiPort, cores, memory,
    masterAddresses, ENDPOINT_NAME, workDir, conf, securityMgr))
  rpcEnv
}
```
worker是如何和master建立联系的？在master消息处理中，receiveAndReply方法第一个处理的消息便是RegisterWorker，也就是处理来自worker的注册消息，那么事情就变得简单了，worker是通过将自己注册到master的RpcEnv中，然后实现通信的。
#### worker注册到master RpcEnv
定位到worker中发送RegisterWorker消息到master Endpoint的方法：
```
private def registerWithMaster() {
  // onDisconnected may be triggered multiple times, so don't attempt registration
  // if there are outstanding registration attempts scheduled.
  registrationRetryTimer match {
    case None =>
      registered = false
      registerMasterFutures = tryRegisterAllMasters()
      connectionAttemptCount = 0
      registrationRetryTimer = Some(forwordMessageScheduler.scheduleAtFixedRate(
        new Runnable {
          override def run(): Unit = Utils.tryLogNonFatalError {
            Option(self).foreach(_.send(ReregisterWithMaster))
          }
        },
        INITIAL_REGISTRATION_RETRY_INTERVAL_SECONDS,
        INITIAL_REGISTRATION_RETRY_INTERVAL_SECONDS,
        TimeUnit.SECONDS))
    case Some(_) =>
      logInfo("Not spawning another attempt to register with the master, since there is an" +
        " attempt scheduled already.")
  }
}
```
其中，masterEndpoint.ask是核心，发送了一个RegisterWorker消息到masterEndpoint并期待对方的RegisterWorkerResponse，对response做出相应的处理。这样worker就成功和master建立了连接，它们之间可以互相发送消息进行通信。ps：其实worker注册到master的步骤有一点复杂，涉及到容错等问题，具体的实现这里暂时不做讨论。
#### worker到master的通信
worker和master之间是一个主从关系，worker注册到master之后，master就可以通过消息传递实现对worker的管理，在worker中有一个方法：
```
private def sendToMaster(message: Any): Unit = {
  master match {
    case Some(masterRef) => masterRef.send(message)
    case None =>
      logWarning(
        s"Dropping $message because the connection to master has not yet been established")
  }
}
```
#### master到worker的通信
master要对worker实现管理也是通过发送消息实现的，比如launchExecutor(worker: WorkerInfo, exec: ExecutorDesc)方法中：
```
private def launchExecutor(worker: WorkerInfo, exec: ExecutorDesc): Unit = {
  logInfo("Launching executor " + exec.fullId + " on worker " + worker.id)
  worker.addExecutor(exec)
  //向worker发送LaunchExecutor消息
  worker.endpoint.send(LaunchExecutor(masterUrl,
    exec.application.id, exec.id, exec.application.desc, exec.cores, exec.memory))
  exec.application.driver.send(
    ExecutorAdded(exec.id, worker.id, worker.hostPort, exec.cores, exec.memory))
}
```
master向worker发送了LaunchExecutor消息告诉worker应该启动executor了，而worker中的receive方法中对LaunchExecutor消息进行处理并完成master交代给自己的任务。
