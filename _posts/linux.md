---
title: linux
tags:
 - linux
categories: 经验分享
---
### 创建用户
```
sudo  useradd  -d  "/home/tt"   -m   -s "/bin/bash"   tt
```
选项：
* -d:指定用户的主目录
* -m:如果存在不再创建，但是此目录并不属于新创建用户 如果主目录不存在，则强制创建 -m & -d一块使用
* -s:指定用户登陆时的shell版本
* -M:不创建主目录

### sudo             cp /usr/share/zoneinfo/Asia/Shanghai /etc/localtime

### ssh-keygen -t rsa

### 使用setup.py方式安装的包该如何卸载

只能手工移除安装的文件，如果你不知道这个安装程序做了什么，可以用--record重新安装一次:

$ python setup.py install --record files.txt
$ cat files.txt | xargs rm -rf          #删除这些文件

### sudo docker container ls

### docker ps 显示正在运行的容器

### echo $? 上条命令的退出码
