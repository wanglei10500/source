---
title: 用hexo搭建个人博客
---
部分的搭建过程在官方文档中已经有详细的介绍，个人只记录在搭建中有出处的地方。
## 安装

### 前提条件
前提需要安装git、node.js
linux版的node.js可从官网下载免安装的版本。[node官网](https://nodejs.org/en/) ，当前描述版本为v4.4.5LTS，将node的bin路径配置在环境变量的PATH中，可直接使用node的node命令和npm命令。

### 建站
根据官网的说明 安装hexo并初始化环境
初始化hexo
```
$ hexo init <folder>   新建一个网站
$ cd <folder>
$ npm install

$ hexo server  启动服务器 默认访问网址为http://localhost:4000,-p 重设端口 
```
### 使用NexT主题

 说明：站点配置文件:网站目录下 _config.yml文件，主题配置文件:主题目录下_config.xml



## 参考链接：
[hexo官网](https://hexo.io/zh-cn/)
[NexT使用文档](http://theme-next.iissnan.com/) 



