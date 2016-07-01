---
title: Linux系统修改默认的TCP keepalive时间
tags:
 - tcp
 - keepalive
categories:经验分享 
---
对于长连接TCP Socket 面对如果需要检查对端是否断线的需求，下面主要描述采用TCP的KEEPALIVE属性检测对端掉线的情况。

### 问题描述

长连接Socket关闭一般有两种情况。
1、连接正常关闭，调用close或shutdown的关闭方式，send与recv立刻返回错误，socket返回SOCK_ERR
2、设备因掉电或断网而异常关闭

对于异常断线的情况，一般有以下两种办法解决
1、定义一种心跳包，定时向对端发心跳包，查看是否有ACK，一般比较通用。
2、使用TCP的keepalive机制

keepalive原理：当server端检测超过一段时间没有传输数据(/proc/sys/net/ipv4/tcp_keepalive_time 7200 默认2小时)，会向client端发送一个keepalive packet ,客户端会有3种反应：
1、连接正常，返回ACK server端收到ACK后重置计时器。
2、client异常关闭，或网络断开，client无响应，在一定时间后重新发送keepalive packet(/proc/sys/net/ipv4/tcp_keepalive_intvl 75) 重发一定次数(/proc/sys/net/ipv4/tcp_keppalive_probes 9)
3、client端以前崩溃，但已经重新启动。server收到的探测响应是一个复位，server端终止连接。


修改系统的3个默认值可以根据我们的需求设置keepalive的时间.


### 临时方法

```
sysctl -w net.ipv4.tcp_keepalive_time=60
sysctl -w net.ipv4.tcp_keepalive_intvl=20
sysctl -w net.ipv4.tcp_keepalive_probes=3
```

### 全局方法
```
修改/etc/sysctl.conf文件 重启后生效
net.ipv4.tcp_keepalive_time=60
net.ipv4.tcp_keepalive_intvl=20
net.ipv4.tcp_keepalive_probes=3
```


