---
title: spark任务调度-ExecutorBackend
tags:
 - spark
categories: 经验分享
---

* 每个	Worker	上存在一个或者多个	ExecutorBackend	进程。每个进程包含一个	Executor	对象,该对象持有一个线程池,每个线程可以执行一个	task。
* 每个	application	包含一个	driver	和多个	executors,每个	executor	里面运行的	tasks	都属于同一个	application。
* 在	Standalone	版本中,ExecutorBackend	被实例化成	CoarseGrainedExecutorBackend	进程。
* Worker	通过持有	ExecutorRunner	对象来控制	CoarseGrainedExecutorBackend	的启停。

### CoarseGrainedExecutorBackend

Executor负责计算任务，即执行task， executor对象的创建和维护是由CoarseGrainedExecutorBackend负责的

CoarseGrainedExecutorBackend在spark运行期是一个单独进程

![spark](http://img.blog.csdn.net/20170408224053499?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvdTAxMTU2NDE3Mg==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

org.apache.spark.executor.CoarseGrainedExecutorBackend

1. CoarseGrainedExecutorBackend是RpcEndpoint的子类，能够和Driver进行RPC通信，其生命周期方法onStart一定要关注，看执行了哪些动作。
2. CoarseGrainedExecutorBackend维护了两个属性executor和driver，executor负责运行task，driver负责和Driver通信。
3. ExecutorBackend有抽象方法statusUpdate，负责将Executor的计算结果返回给Driver。
4. CoarseGrainedExecutorBackend是spark运行期的一个进程，Executor运行在该进程内。

#### 启动CoarseGrainedExecutorBackend

.. TODO Launch Executor 后

Worker进程收到LaunchExecutor消息，Worker将收到的消息封装为ExecutorRunner对象，调用start方法（org.apache.spark.deploy.worker.ExecutorRunner）start方法启动后，调用ExecutorRunner的fetchAndRunExecutor方法

fetchAndRunExecutor方法中将收到的信息拼接为Linux命令，然后使用ProcessBuilder执行Linux命令启动CoarseGrainedExecutorBackend，和启动Driver的方式如出一辙

会调用CoarseGrainedExecutorBackend的main方法，main方法中处理命令行传入的参数，然后创建RpcEnv，并注册CoarseGrainedExecutorBackend(org.apache.spark.executor.CoarseGrainedExecutorBackend)

将CoarseGrainedExecutorBackend注册到RpcEnv会调用其onStart方法

#### 小结
![spark](http://img.blog.csdn.net/20170409001714089?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvdTAxMTU2NDE3Mg==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
