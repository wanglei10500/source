---
title: YARN
tags:
 - hadoop
 - YARN
categories: 经验分享
---

![Yarn](https://www.iteblog.com/pic/hadoop/The_YARN_architecture_iteblog.jpg)

Hadoop 1.x JobTracker 负责管理集群的资源，作业调度、作业监控。

### ResourceManager
每个Hadoop集群有一个ResourceManager(如果是HA有两个，有一个时active状态)，负责整个集群的计算资源，将资源分别给应用程序。

ResourceManager内部主要两个组件:
* Scheduler: 插拔式，可根据需求实现不同的调度器，Yarn提供了FIFO、容量、公平调度器。唯一功能就是给提交到集群的应用程序分配资源，对可用的资源和运行的队列进行限制。Scheduler并不对作业进行监控。
* ApplicationsManager(AsM):管理集群应用程序application masters。负责接收应用程序的提交；为application master启动提供资源;监控应用程序的运行进度以及在应用程序故障重启它。
### NodeManager
Yarn每个节点上的代理，管理Hadoop集群单个计算节点

NodeManager定期向ResourceManager发送心跳来更新其健康状态。 监督Container的生命周期，监控每个Container的资源使用情况，追踪节点健康状况，管理日志和不同应用程序的附属服务(auxiliary service)。
### ApplicationMaster
应用程序级别，每个ApplicationMaster管理运行在Yarn上的应用程序。Yarn将ApplicationMaster看作第三方组件。

负责和ResourceManager Scheduler协商资源，并和NodeManager通信来运行相应的Task。

ApplicationMaster也会追踪应用程序的状态，监控容器的运行进度，当容器运行完成，ApplicationMaster会向ResourceManager注销这个容器

整个作业完成，向ResourceManager注销自己，这样资源可以分配给其他应用程序
### Container
 与特定节点绑定的，包含了内存、CPU 磁盘等逻辑资源。 容器实现中，容器只包括勒内存和CPU。

 容器是ResourceManager Scheduler服务动态分配的资源构成。容器授予ApplicationMaster使用特定主机的特定数量资源的权限。

 ApplicationMaster也是在容器中运行的，在应用程序分配的第一个容器中运行。
