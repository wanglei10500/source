---
title: Ubuntu下Airflow搭建及应用
tags:
 - airflow
categories: 环境搭建
---

## Airflow组件
Airflow被Airbnb内部用来创建、监控和调整数据管道。任何工作流都可以在这个使用Python来编写的平台上运行。
在一个可扩展的生产环境中，Airflow含有以下组件：

1. 一个元数据库（MySQL或Postgres）

2. 一组Airflow工作节点

3. 一个调节器（Redis或RabbitMQ）(可省略)

4. 一个Airflow Web服务器

Airflow提供一个非常容易定义DAG的机制：一个开发者使用Python 脚本定义他的DAG。
然后自动加载这个DAG到DAG引擎，为他的首次运行进行调度。修改一个DAG就像修改Python 脚本一样容易。

## Airflow特点 与Azkaban比较
* 动态性：Airflow pipeline是以python代码形式做配置
* 扩展性:可以自定义operator(运算子), 有几个executor(执行器)可供选择
* 灵活性:基于jinja模板引擎很容易做到脚本命令参数化
* 扩展性:模块化的结构, 加上使用了消息对列来编排多个worker(启用CeleryExcutor), airflow可以做到无限扩展.

linkedin的Azkaban 使用java properties文件维护上下游关系, 任务资源文件需要打包成zip, 部署不是很方便.airflow可方便的设置任务失败重试，任务执行统计图表等比azkaban要合理和完善。

## Airflow环境搭建
所依赖python环境应为python2.7.X
建议升级系统环境、确保apt-get gcc均为最新

```
sudo apt-get update
sudo apt-get upgrade gcc
# 安装python2.7 dev
sudo apt-get install python2.7-dev
```

### PIP
pip类似RedHat里面的yum，ubuntu下可使用命令安装pip
```
sudo apt-get install python-pip
```

### 使用pip安装airflow
```
# airflow needs a home, ~/airflow is the default,
export AIRFLOW_HOME=~/airflow

# install from pypi using pip
pip install airflow

# initialize the database
airflow initdb


# start the web server, default port is 8080
airflow webserver -p 8080
```
### 修改数据库为mysql
airflow默认数据库为sqlite，生产环境中若要改为mysql,首先需要安装python连接mysql库 ，应该使用 mysqlclient 包，mysql-connector-python 包会报错。
修改airflow.cfg文件
```
sudo apt-get install libmysqld-dev
sql_alchemy_conn = mysql://root:mysql1@127.0.0.1:3306/airflow
```

###  修改Executor
有三个 Executor 可供选择, 分别是: SequentialExecutor 和 LocalExecutor 和 CeleryExecutor, SequentialExecutor仅仅适合做Demo(搭配Sqlite backend), LocalExecutor 和 CeleryExecutor 都可用于生产环境, CeleryExecutor 将使用 Celery 作为Task执行的引擎, 扩展性很好, 当然配置也更复杂, 需要先setup Celery的backend(包括RabbitMQ, Redis)等. 其实真正要求扩展性的场景并不多, 所以LocalExecutor 是一个很不错的选择了.

## Airflow所用组件的简介
### SqlAlchemy
SQLAlchemy 是python 操作数据库的一个库。能够进行 orm 映射，SQLAlchemy“采用简单的Python语言，为高效和高性能的数据库访问设计，实现了完整的企业级持久模型”
### Celery
Celery 是一个简单、灵活且可靠的，处理大量消息的分布式系统，并且提供维护这样一个系统的必需工具。
它是一个专注于实时处理的任务队列，同时也支持任务调度。
### Cron
Cron表达式是一个字符串，字符串以5或6个空格隔开，分为6或7个域，每一个域代表一个含义，Cron有如下两种语法格式：
Seconds Minutes Hours DayofMonth Month DayofWeek Year或
Seconds Minutes Hours DayofMonth Month DayofWeek
### Jinja2
Jinja2是Python下一个被广泛应用的模版引擎，他的设计思想来源于Django的模板引擎，并扩展了其语法和一系列强大的功能。其中最显著的一个是增加了沙箱执行功能和可选的自动转义功能，这对大多应用的安全性来说是非常重要的。

## Airflow的应用
### DAG脚本的开发
目前应用场景是使用Airflow进行spark任务的调度，只需要开发一个提交任务的Python脚本作为DAG，定时提交任务即可，需要的操作库为BashOperator。
使用jinja模板将容易改变的参数以配置形式传入命令中。

将这个.py文件放在 AIRFLOW_HOME目录下的dags文件夹下即可
```
airflow resetdb
```
### 开启调度器
```
airflow scheduler  
```
