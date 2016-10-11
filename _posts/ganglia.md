---
title: Ubuntu下 ganglia的搭建
tags:
 - ganglia的搭建
categories: 环境搭建
---
Ganglia是一个跨平台可扩展的，高性能计算系统下的分布式监控系统，如集群和网格。

## 搭建环境
Ubuntu14.04
安装gmetad的机器：10.0.8.162（Spark-Master）

安装gmond的机器：10.0.8.162（Spark-Master），10.0.8.81（Spark-Slave1），10.0.8.177（Spark-Slave2），10.0.8.178（Spark-Slave3），10.0.8.179（Spark-Slave4），10.0.8.190（Spark-Slave5），10.0.8.54（Spark-Slave6）

浏览监控web页面的机器：10.0.8.162（Spark-Master）
## Ganglia简介

Ganglia 监控套件包括三个主要部分：gmond，gmetad，ganglia-web。

gmond 是一个守护进程，他运行在每一个需要监测的节点上，收集监测统计，发送和接受在同一个组播或单播通道上的统计信息。

gmetad 也是一个守护进程，他定期检查gmonds ，从那里拉取数据，并将他们的指标存储在RRD存储引擎中。它可以查询多个集群并聚合指标。RRD也被用于生成用户界面的web前端。

ganglia-web 他应该安装在有gmetad运行的机器上，以便读取RRD文件。

## Ubuntu上安装配置LAMP
* Linux，操作系统；
* Apache，web服务器；
* MySQL，数据库；
* PHP，动态脚本语言；

###  安装Apache
Apache是一个Web服务器软件，它在Ubutnu的默认软件仓库中，简单的执行如下命令安装：
```
$ sudo apt-get update
$ sudo apt-get install apache2

# Apache服务的启动、停止、查询状态命令
$ sudo service apache2 start
$ sudo service apache2 stop
$ sudo service apache2 status
```
### 安装Mysql
```
$ sudo apt-get install mysql-server php5-mysql
```
mysql-server是mysql主服务程序；php5-mysql是mysql和PHP的沟通桥梁，通过它，php脚本可以直接连接mysql。
```
# 创建mysql数据库目录结构
$ sudo mysql_install_db
# 执行如下安全脚本，提要系统安全
sudo mysql_secure_installation

# Mysql服务的启动、停止、查询状态命令

$ sudo service mysql start
$ sudo service mysql stop
$ sudo service mysql status

```
### 安装PHP
PHP （“PHP: Hypertext Preprocessor”，超文本预处理器的字母缩写）是一种被广泛应用的开放源代码的多用途脚本语言，它可嵌入到HTML中，尤其适合web 开发。
```
$ sudo apt-get install php5 libapache2-mod-php5 php5-mcrypt
# 配置Apache，添加index.php 将index.php放在DirectoryIndex的第一位 优先处理
$ sudo vim /etc/apache2/mods-enabled/dir.conf
# 重启Apache
$ sudo service apache2 restart
```
### 测试LAMP环境
在网站根目录创建文件info.php，在Ubuntu 14.04上，网站根目录是/var/www/html/。
```
# 编辑文件
$ sudo vim /var/www/html/info.php
# 添加如下内容
<?php
phpinfo();
?>
# 保存退出
# 访问http://your_server_IP_or_domain/info.php
```
这个页面列出了服务器的基本信息。这在调试和检查设置时很有用。

## Ganglia安装
首先确保在 Ubuntu14.04 上安装了 LAMP 服务
```
# master上
sudo apt-get install ganglia-monitor rrdtool gmetad ganglia-webfrontend
# 安装过程会询问是否重新启动apache，选择yes

# slave上
$ sudo apt-get install ganglia-monitor
```
## Ganglia配置
复制Ganglia web前端apache配置文件
```
$ sudo cp /etc/ganglia-webfrontend/apache.conf /etc/apache2/sites-enabled/ganglia.conf
```
编辑 ganglia meta daemon配置文件：
```
$ sudo vim /etc/ganglia/gmetad.conf
更改如下
data_source "MySpark" 10 10.0.8.162:8649 10.0.8.81:8649 10.0.8.177:8649 10.0.8.178:8649 10.0.8.179:8649 10.0.8.190:8649 10.0.8.54:8649
```
配置为单播：可以跨网段传播，只将信息发送给指定的机器。要配置成为单播你应该指定一个（或者多个）接受的主机。

编辑节点配置文件：
```
$ sudo vim /etc/ganglia/gmond.conf
修改内容
cluster {
  name = "MySpark"
  owner = "ganglia"
  latlong = "unspecified"
  url = "unspecified"
}
udp_send_channel {
  # mcast_join = 239.2.11.71
  host = 10.0.8.162
  port = 8649
  ttl = 1
}
udp_recv_channel {
  # mcast_join = 239.2.11.71
  port = 8649
  # bind = 239.2.11.71
}

```



### 启动Ganglia
```
$ sudo /etc/init.d/ganglia-monitor start
$ sudo /etc/init.d/gmetad start
$ sudo /etc/init.d/apache2 restart
```

访问ganglia web前端，http://ip:port/ganglia/

### 参考资料
https://github.com/ganglia

http://blog.topspeedsnail.com/archives/3049

http://www.jianshu.com/p/e41137aa6f43
