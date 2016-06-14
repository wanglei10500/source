---
title: Linux下 rabbit MQ单机环境和集群环境的搭建
tags:
 - rabbit MQ
categories: 环境搭建
---
本次搭建版本为rabbitmq-server-generic-unix-3.6.2
## Erlang的安装
由于rabbit MQ基于Erlang。所以必须先安装Erlang环境。
Ubuntu下可直接用sudo apt-get install Erlang

## simplejson安装
simplejson是依赖python脚本，所以必须有python环境。
[simplejson下载地址](https://pypi.python.org/pypi/simplejson/_)
```
tar -zxvf simplejson-x.x.x 
cd simplejson-x.x.x
python setup.py install
```
## rabbit mq安装
[rabbit mq下载地址](https://www.rabbitmq.com/download.html)
下在linux版tar.xz后缀的文件
解压后 可在/sbin目录下，运行单机版rabbit。
```
cd rabbitmq/sbin  ./rabbitmq-server -detached实现后台启动

./rabbitmqctl stop 关闭rabbitmq

执行./rabbitmq-plugin enable rabbitmq_management 开启管理界面。默认地址：127.0.0.1:15672 
```

##  解决无法远程连接的问题
rabbit默认帐号guest只能通过localhost登陆使用，官方文档对此的描述为"guest" user can only connect via localhost  所以需要添加新的用户
```
增加一个用户 	 ./rabbitmqctl add_user Username Password
删除一个用户 	 ./rabbitmqctl delete_usr Username
修改用户的密码   ./rabbitmqctl change_password Username Newpassword
查看当前用户列表 ./rabbitmqctl list_users
用户角色	 ./rabbitmqctl set_user_tags Username Tag
		  (Tag 可设置为管理员administrator)
用户权限设置     ./rabbitmqctl set_permissions -p / Username ".*" ".*" ".*" 
```
## 集群环境搭建

rabbit MQ 的集群依赖于erlang的集群来工作，所以必须构建起erlang的集群环境。erlang集群中的各节点通过一个cookie文件来实现的。这个cookie文件存放在$HOME/.Erlang.cookie。文件权限应该是400.必须保证各个节点的cookie保持一致。
将其中一台节点的值复制下来保存到其他节点上。
```
$cat .erlang.cookie
先更改文件权限，给予其可写权限
$chmod 777 .erlang.cookie
$echo -n "XXXXXXXXXX" >$HOME/.erlang.cookie
$chmod 400 .erlang.cookie
```
此步骤建议在rabbit 启动前完成
保证主机之间可以互相解析
修改各节点主机的/etc/hosts文件，以确保各个计算机可以互相解析
10.0.8.1 n1
10.0.8.2 n2
10.0.8.3 n3
准备工作已经完成，集群构建只需把其他节点添加到一台节点中即可，这里采用将n2 n3添加到n1上，无特殊说明下面的步骤在n2 n3上都要做。
```
$cd /rabbitmq/sbin
$./rabbitmq-server -detached //三个结点都启动
$./rabbitmqctl status //查看状态
$./rabbitmqctl stop_app //关闭节点
$./rabbitmqctl reset

```
在n2上：
```
$./rabbitmqctl join_cluster n1
```
在n3上：
```
$./rabbitmqctl join_cluster n1
$./rabbitmqctl start_app
```
开启集群的所有节点，可进入rabbit的web管理界面查看集群是否正常运行
