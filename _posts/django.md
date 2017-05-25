---
title: oss
tags:
 - pip
 - python
categories: 经验分享
---

### 查看python版本号
```
$ python -c "import django; print(django.get_version())"
```

### 创建django项目
```
cd到保存代码的目录
$ django-admin startproject mysite

mysite/
    manage.py
    mysite/
        __init__.py
        settings.py
        urls.py
        wsgi.py
```
外层mysite/为项目的容器，可重命名为任何名字

manage.py 命令行工具

内层mysite/是项目真正Python包 是导入任何东西时需要使用的Python包名字

settings.py 配置文件

urls.py url声明

wsgi.py 项目与WSGI兼容的Web服务器入口

## django-admin

### settings

#### 数据库配置
```
settings中DATABASES

sqlite
'default': {
        'ENGINE': 'django.db.backends.sqlite3',
        'NAME': os.path.join(BASE_DIR, 'db.sqlite3'),
    }
mysql    
'default': {
    'ENGINE': 'django.db.backends.mysql',
    'NAME': 'django',
    'USER':'root',
    'PASSWORD':'mysql1',
    'HOST':'10.0.8.162',
    'PORT':3306,
}    
```
#### 时区
```
TIME_ZONE
默认：‘America/Chicago’ 新的模板为’UTC’  	Asia/Shanghai
USE_TZ
默认：True 它是Django 显示模板中以及解释表单中的日期和时间默认使用的时区。
     False 它将成为Django 存储所有日期和时间时使用的时区。
```

#### INSTALLED_APPS
```
默认包含
django.contrib.admin  管理站点
django.contrib.auth   认证系统
django.contrib.contenttypes 用于内容类型的框架
django.contrib.sessions 会话框架
django.contrib.messages 消息框架
django.contrib.staticfiles 管理静态文件的框架
```

###  创建数据库表
```
根据mysite/settings.py文件中的数据库设置创建 包括迁移
python manage.py migrate
```

### 开发服务器
```
$ python manage.py runserver
$ python manage.py runserver 0.0.0.0:8000
```
### 创建模型
mysite 是项目，创建应用
```
$ python manage.py startapp polls
```
应用文件夹下的models.py文件中创建数据库模型
```
from django.db import models
class Question(models.Model):
    question_text = models.CharField(max_length=200)
    pub_date = models.DateTimeField('date published')


class Choice(models.Model):
    question = models.ForeignKey(Question)
    choice_text = models.CharField(max_length=200)
    votes = models.IntegerField(default=0)

```
每个字段通过Field类的一个实例表示

### 激活模型
```
在mysite.settings.py 修改INSTALLED_APPS包含polls

$ python manage.py makemigrations polls 创建迁移文件
```
模型文件变更三个步骤
1. 修改你的模型(models.py)
2. python manage.py makemigrations 为这些修改创建迁移文件
3. python manage.py migrate

### __str__ 与 __unicode__
Python3中 只需使用__str__()

Python2中 定义__unicode__方法返回Unicode值。Django模型具有一个默认的__str__()方法，调用__unicode__（）并将结果转换为UTF-8字节字符串

### 创建管理员账户
```
$ python manage.py createsuperuser
```
### 启动开发服务器
```
$ python manage.py runserver   http://127.0.0.1:8000/admin/
```
#### 在管理站点可编辑应用
```
polls/admin.py
admin.site.register(Question)
```
自定义管理表单
```
class QuestionAdmin(admin.ModelAdmin):
    fields = ['pub_date','question_text']

admin.site.register(Question,QuestionAdmin)
```
表单分割字符集
```
class QuestionAdmin(admin.ModelAdmin):
    fieldsets = [
        (None,{'fields':['question_text']}),
        ('Date information',{'fields':['pub_date'],'classes':['collapse']})
    ]
admin.site.register(Question,QuestionAdmin)
```
