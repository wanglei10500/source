---
title: Linux下 ftp服务器的搭建
tags:
 - ftp
 - vsftpd
categories: 环境搭建
---


### 安装ftp
```
which vsftpd #检查是否已经安装了vsftpd

sudo apt-get install vsftpd
```

### 更改启动状态
```
sudo service vsftpd start # 开启FTP服务

service vsftpd status #查看FTP的状态

sudo service vsftpd stop # 停止FTP服务

sudo service vsftpd restart #重启FTP服务


```

### vsftpd.conf配置
```
ftp默认监听21端口

配置文件默认在/etc/vsftpd.conf
修改配置文件
anonymous_enable=YES #允许匿名用户

local_root=/srv/ftp/ #添加此配置 设置ftp根目录
```
