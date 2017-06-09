---
title: ResourceManager
tags:
 - hadoop
 - ResourceManager
categories: 经验分享
---
## ResourceManager

![ResourceManager](http://dongxicheng.org/wp-content/uploads/2013/02/Hadoop-YARN-infrastructure.jpg)
![ResourceManager](https://2xbbhjxc6wk3v21p62t8n4d4-wpengine.netdna-ssl.com/wp-content/uploads/2012/08/resource_manager_small.gif)


ResourceManager主要由几个部分组成：
### 用户交互
Yarn针对普通用户、管理员、WEB 提供三种对外服务。ClientRMService、AdminService和WebApp:
#### ClientRMService
为普通用户提供的服务，处理来自客户端各种RPC请求，比如提交应用程序、终止应用程序，获取应用程序的运行状态。
#### AdminService
YARN为管理员提供了一套独立的服务接口，防止大量普通用户请求使管理员命令饿死。管理员通过这些接口管理集群,动态更新节点列表，更新ACL列表，更新队列信息等。
#### WebApp
更加友好的展示集群资源使用情况和应用运行状态

### NM管理(manage NMs)
#### NMLivelinessMonitor
监控NM是否活着，如果一个NodeManager在一定时间(默认10min)内未汇报心跳信息。则认为它死掉了，会将其在集群中移除。
#### NodesListManager
维护正常节点和异常节点列表，管理exlude(黑名单)inlude(白名单)节点列表，这两个列表均是在配置文件中设置的，可以动态加载。
#### ResourceTrackerService
处理来自NodeManager的请求，主要包括两种请求：注册和心跳。
其中，注册是NodeManager启动时发生的行为，请求包中包含节点ID、可用资源上限等信息。
心跳是周期性行为，包含各个Container运行状态、运行的Application列表、节点健康状态(可通过脚本设置)，
而ResourceTrackerService则为NM返回待释放的Container列表、Application列表等。

### AM管理(manage AMs)
#### AMLivelinessMonitor
监控AM是否活着，如果一个ApplicationMaster在一定时间(默认10min)内未汇报心跳信息，则认为它死掉了，它上面所有正在运行的Container将被认为死亡，AM本身会被重新分配到另一节点上(用户可指定每个ApplicationMaster的尝试次数)默认一次。
#### ApplicationMasterLauncher
与NodeManager通信，要求它为某个应用程序启动
#### ApplicationMasterService
处理来自ApplicationMaster的请求，主要包括两种请求：注册和心跳。
其中，注册是ApplicationMaster启动时发生的行为，包括请求包中包含所在节点，RPC端口号和tracking URL等信息。
而心跳是周期性行为，包含请求资源的类型描述、待释放的Container列表等，而AMS则为之返回新分配的Container、失败的Container等信息。

### Application管理
#### ApplicationACLsManager
管理应用程序访问权限，包含两部分权限:查看和修改。
查看主要是查看应用程序基本信息，修改主要是修改应用程序优先级、杀死应用程序等。
#### RMAppManager
管理应用程序的启动和关闭。
#### ContainerAllocationExpirer
YARN不允许AM获得Container后长期不对其使用，这会降低整个集群的利用率。当AM收到RM新分配的一个Container后，必须在一定时间(10min)内在对应的NM上启动该Container，否则，RM会回收Container。

### 安全管理
ResourceManager自带了非常全面的权限管理机制，由ClientToAMSecretManager、ContainerTokenSecretManager、ApplicationTokenSecretManager等模块完成。

### 资源分配
#### ResourceScheduler
ResourceScheduler是资源调度器，它按照一定的约束条件(比如 队列容器限制等)将集群中的资源分配给各个应用程序。
ResourceScheduler是一个插拔式模块，默认是FIFO 还提供了Fair Scheduler和Capacity Scheduler两个多租户调度器。
