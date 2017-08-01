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
### 启用TLS
确认服务器安装了openssl
```
sudo apt-get install openssl
```
生成SSL / TLS证书和私钥
```
openssl req -x509 -nodes -keyout /etc/ssl/private/vsftpd.pem -out /etc/ssl/private/vsftpd.pem -days 365 -newkey rsa:2048

req – 是X.509证书签名请求（CSR）管理的命令。
x509 – 表示X.509证书数据管理。
days – 定义证书有效期的天数。
newkey – 指定证书密钥处理器。
rsa：2048 – RSA密钥处理器，将生成一个2048位的私钥。
keyout – 设置密钥存储文件。
out设置证书存储文件，请注意，证书和密钥都存储在同一个文件中： /etc/ssl/private/vsftpd.pem 。

```

修改 /etc/vsftpd.conf配置文件
```
#修改以下配置
rsa_cert_file=/etc/ssl/private/vsftpd.pem

rsa_private_key_file=/etc/ssl/private/vsftpd.pem
#增加以下配置
ssl_enable=YES

ssl_tlsv1=YES

ssl_sslv2=NO

ssl_sslv3=NO

strict_ssl_read_eof=YES

require_ssl_reuse=NO
#当选项require_ssl_reuse设置为YES时，所有SSL数据连接都需要显示SSL会话重用; 证明他们知道与控制信道相同的主秘密。
```
sudo service vsftpd restart #重启FTP服务

### 开启本地虚拟用户的写权限

修改 /etc/vsftpd.conf配置文件
```
local_enable=YES

write_enable=YES

local_umask=022

```

sudo service vsftpd restart #重启FTP服务
