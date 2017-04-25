---
title: python 包管理工具的使用
tags:
 - pip
 - python
categories: 经验分享
---

python两个包管理工具easy_install和pip，python2.7中easy_install默认安装。

## install on ubuntu
```
sudo pip install python-pip
```
## pip的使用

### Upgrade pip:
```
pip install -U pip

```
### install a package:
```
pip install SomePackage
安装特定版本package，使用==,>=,<=,>,<指定版本号
pip install SomePackage==1.0.4
pip install 'Markdown>2.0,<2.0.3'
如果有requirement.txt 直接pip install -r requirement.txt

```
### Uninstall package:
```
pip uninstall SomePackage
```

### upgrade a package:
```
pip install --upgrade SomePackage
```
### 查看软件包安装了哪些文件及路径信息:
```
pip show --files SomePackage
```
### 查看哪些包有更新版本:
```
pip list --outdated
```
### 查询软件包
```
搜索与关键词相关的软件包
pip search 'airflow'
```
### 列出所有安装的软件包
```
pip list
```
### 生成requirements.txt文件 针对环境
```
pip airflow > requirements.txt
```
### 安装requirements.txt依赖
```
pip install -r requirements.txt
```
