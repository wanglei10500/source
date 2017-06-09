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
NodeManager自身健康状态机制(NodeHealthCheckerService) 汇报给ResourceManager，然后ResourceManager端通过RMNodeEventType.STATUS_UPDATE更新NodeManager的状态。

这种内置的健康检测机制主要有两种:
#### 健康状态检测脚本
每个NodeManager上都有一个名为NodeHealthScriptRunner的类,其会启动一个NodeHealthMonitor-Timer的Timer(默认10min，yarn.nodemanager.health-checker.interval-ms)

用户编写脚本通过yarn.nodemanager.health-checker.script.path指定。以下几种结果则认为处于不健康状态：
* 脚本输出包含ERROR开头的字符串
* 执行脚本的时候出现超时
* 执行脚本出现异常
如果出现ExitCodeException异常，并不认为NodeManager节点健康出现异常。

NodeHealthScriptRunner类涉及的参数:
* yarn.nodemanager.health-checker.script.path : Hadoop集群管理员编写的脚本路径，如果没有设置，NodeManager不会启动NodeHealthScriptRunner
* yarn.nodemanager.health-checker.interval-ms : 每隔多久执行健康脚本 默认10min
* yarn.nodemanager.health-checker.script.timeout-ms : 执行健康检测脚本超过了这个属性配置的时间，则认为节点不健康。默认值是20分钟
* yarn.nodemanager.health-checker.script.opts : 健康检测脚本的输入参数，如果有多个请使用空格分割

好处：除内存核CPU，其他的比如磁盘使用、系统负载、网络等主动告诉ResourceManager自身的监控状态。

demo：内存使用率达到95%以上，认为节点不健康
```
#!/bin/bash

echo $1
echo $2

mem_usage=$(echo `free -m | awk '/^Mem/ {printf("%u", 100*$3/$2);}'`)
echo "Usage is $mem_usage%"
if [ $mem_usage -ge 95 ] then
  echo 'ERROR: Memory Usage is greater than 95%'
else
  echo 'NORMAL: Memory Usage is less than 95%'
fi
```
NodeManager 进行如下配置：
```
<property>
  <name>yarn.nodemanager.health-checker.script.path</name>
  <value>/user/iteblog/check_memory_usage.sh</value>
</property>

<property>
  <name>yarn.nodemanager.health-checker.script.opts</name>
  <value>第一个参数 第二个参数</value>
</property>
```


### 本地目录健康检测
检测磁盘目录主要是yarn.nodemanager.local-dirs 和 yarn.nodemanager.log-dirs 参数指定的目录

这两个目录分别用于存储应用程序运行的中间结果(比如 MapReduce作业中Map Task 的中间输出结果)和日志文件存放目录接表。

两个参数都可以配置成多个目录，逗号分隔。如果两个参数配置的目录不可用比例达到一定的设置，则认为不健康。

不可用的定义是：运行NodeManager节点的进程是否 可读、可写、可执行。不健康被放入failedDirs列表里面。

本地目录健康检测主要涉及几个参数：
* yarn.nodemanager.disk-health-checker.interval-ms : 执行频率，默认2min。
* yarn.nodemanager.disk-health-checker.enable : 是否启用本地目录健康检测，默认启动。
* yarn.nodemanager.disk-health-checker.min-healthy-disks : 正常目录数目相对总目录总数的比例，低于这个值认为不正常，默认值0.25。.

两种检测机制随着NodeManager节点启动而运行，检测到的状态随心跳信息发到ResourceManager端。
