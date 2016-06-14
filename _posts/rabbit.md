---
title: Linux下 rabbit MQ单机环境和集群环境的搭建
tags:
 - rabbit MQ
categories: 环境搭建
---
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
由于默认帐号guest只能通过localhost登陆使用，所以需要添加新的用户
```
增加一个用户 	 ./rabbitmqctl add_user Username Password
删除一个用户 	 ./rabbitmqctl delete_usr Username
修改用户的密码   ./rabbitmqctl change_password Username Newpassword
查看当前用户列表 ./rabbitmqctl list_users
用户角色	 ./rabbitmqctl set_user_tags Username Tag
		  (Tag 可设置为管理员administrator)
用户权限设置     ./rabbitmqctl set_permissions -p / Username ".*" ".*" ".*" 
```

