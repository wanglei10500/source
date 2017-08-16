---
title: jvm命令
tags:
 - jvm
 - java
categories: 经验分享
---

### jvm命令

#### -verbose:gc
-verbose:gc 表示输出虚拟机中GC的详细情况

#### -Xmx
最大堆大小 默认：物理内存的1/4(<1GB)
#### -Xms
初始堆大小 默认：物理内存的1/64(<1GB) -Xms与-Xmx相同则不可扩展
#### -Xmn
年轻代大小
#### -XX:+HeapDumpOnOutOfMemoryError
-XX:+HeapDumpOnOutOfMemoryError 虚拟机在出现内存溢出异常时Dump出当前的内存堆存储快照以便事后进行分析
-XX:HeapDumpPath=/home/wl/gc.hprof dump文件存储位置
#### -XX:+PrintGCDetails
-XX:+PrintGCDetails
#### -Xss
-Xss 每个线程的栈容量
