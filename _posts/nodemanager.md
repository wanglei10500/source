---
title: NodeManager
tags:
 - hadoop
 - NodeManager
categories: 经验分享
---

ResourceManager内维护了NodeManager的生命周期 。每一个NodeManager中都有一个RMNode与其对应。还定义了NodeManager的状态(status)以及触发状态转移的事件(event)。

![NodeManager](https://www.iteblog.com/pic/hadoop/NodeManager_life_cycle_iteblog.png)

#### org.apache.hadoop.yarn.server.resourcemanager.rmnode.RMNode
接口 NodeManager---RMNode 对应。维护NodeManager的可用资源(内存、CPU)和静态信息(NodeManager的ID、hostname、Http端口、健康状况、机架名称等)。
#### org.apache.hadoop.yarn.server.resourcemanager.rmnode.RMNodeImpl
实现RMNode 。记录当前NodeManager中所有运行的applications/Containers 。定义了NodeManager的状态转移以及处理的类。
#### org.apache.hadoop.yarn.server.resourcemanager.rmnode.RMNodeEventType
枚举类 定义NodeManager所有的事件类型
#### org.apache.hadoop.yarn.api.records.NodeState
枚举类 定义NodeManager所有可能的状态

### NodeManager状态
开始状态:NEW
最终状态： DECOMMISSION / REBOOTED / LOST

向ResourceManager注册的初始化都为NodeState.NEW。注册成功其状态会更新为NodeState.RUNNING，ResourceManager做两件事：
* 如果这个NodeManager之前在ResourceManager中的inactive节点列表里面，说明之前处于不正常状态。需要把该NodeManager从inactive节点列表移除，并更新集群的Metrics信息（增加ACTIVE NODE个数 减少异常节点个数）
* 否则直接更新集群Metrics信息，增加Active Node的个数

NodeManager在启动之后会默认每隔1s(yarn.resourcemanager.nodemanagers.heartbeat-interval-ms) 向ResourceManager发送心跳信息

ResourceManager端会启动PingChecker线程默认200s（yarn.nm.liveness-monitor.expiry-interval-ms 参数值得三分之一）检测所有注册到 ResourceManager 的节点

一旦发现有节点超过 600s没有发送心跳信息，则认为这个节点出问题了,从running中移除并发送RMNodeEventType.EXPIRE事件

RMNodeImpl接收到事件通过DeactivateNodeTransition类来处理。 将节点移除加入到inactiveNodes列表里面

状态转移到NodeState.LOST 标记为NodesListManagerEventType.NODE_UNUSABLE。所有这个节点上的Container会标记为失败，分配到新的NodeManager上运行。

### NodeManager自身状态监测
