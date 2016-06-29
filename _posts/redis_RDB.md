---
title:  解决Redis错误MISCONF Redis is configured to save RDB snapshots...
tags:
 - redis
categories: 问题解决
---

在Redis的使用过程中，在更新redis数据时出现了错误信息。
可能出现的错误信息：MISCONF Redis is configured to save RDB snapshots, but is currently not able to persist on disk. Commands that may modify the data set are disabled. Please check Redis logs for details about the error.

由于Redis默认设置stop-writes-on-bgsave-error yes

默认情况下，如果在RDB snapshots持久化过程中出现问题，设置该参数后，Redis是不允许用户
进行任何更新操作(set...)。

解决方法：
```
./redis-cli
config set stop-writes-on-bgsave-error no
```
