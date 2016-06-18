---
title: 用virtualenv打造干净的python环境
tags:
 - python
 - virtualenv
categories: 经验分享
---

由于一些时候需要用到不同版本python库等应用情况，virtualenv可以为我们创建一个干净的python环境，不包含任何第三方库。
python中所有的第三方库都会被安装在site-packages目录下。

### 1、需要安装virtualenv，如果没有安装pip，ubuntu可以通过sudo apt-get install python-pip来安装
```
pip install virtualenv
或 sudo apt-get install virtualenv
```

### 2、创建工程目录:
```
mkdir PyProjects
```

### 3、进入工程目录,创建独立的python运行环境，命令为venv：
```
virtualenv --no-site-packages venv
```

### 4、运行虚拟环境
```
source venv/bin/activate
```

### 5、退出当前的venv环境
```
deactivate
```
