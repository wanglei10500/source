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
#### -verbose:class
-verbose:class 可查看加载类的情况
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
#### -XX:PermSize
-XX:PermSize 设置持久代(perm gen)初始值  默认值:物理内存的1/64
#### -XX:MaxPermSize
-XX:MaxPermSize 设置持久代的最大值 默认值:物理内存的1/4
#### -Xnoclassgc
-Xnoclassgc 表示不对class进行垃圾回收，如果使用Spring、hibernate这些框架用到很多反射，产生大量临时的class、不能打开
#### -XX:+TraceClassLoading
-XX:+TraceClassLoading 用来打印类被加载的过程信息，用来诊断应用的内存泄露问题
#### -XX:+TraceClassUnloading
-XX:+TraceClassUnloading 用来打印类卸载的过程信息
#### -XX:SurvivorRatio
-XX:SurvivorRatio Eden区与Survivor区的大小比值 设置为8,则两个Survivor区与一个Eden区的比值为2:8,一个Survivor区占整个年轻代的1/10
#### -XX:PretenureSizeThreshold
-XX:PretenureSizeThreshold 对象超过多大直接在旧生代分配 单位字节 新生代采用Parallel Scavenge GC时无效 大于这个设置值的对象直接在老年代分配，避免Eden区及两个Survivor区之间发生大量的内存复制 只对Serial和ParNew两款收集器有效
#### -XX:+UseConcMarkSweepGC
-XX:+UseConcMarkSweepGC	使用CMS内存收集
#### -XX:+UseParNewGC
-XX:+UseParNewGC 设置年轻代为并行收集
#### -XX:ParallelGCThreads
-XX:ParallelGCThreads 并行收集器的线程数 适用于CMS
#### -XX:MaxGCPauseMillis
-XX:MaxGCPauseMillis 每次年轻代垃圾回收的最长时间(最大暂停时间) 若无法满足此时间JVM会自动调整年轻代大小，以满足此值

允许设置一个大于0的毫秒数 GC停顿时间缩短是牺牲吞吐量和新生代空间换取的
#### -XX:GCTimeRatio
-XX:GCTimeRatio 	设置垃圾回收时间占程序运行时间的百分比 公式为1/(1+n)

参数的值应当是一个大于0小于100的整数 默认值99 允许最大1%的垃圾收集时间
#### -XX:+UseAdaptiveSizePolicy
-XX:+UseAdaptiveSizePolicy 自动选择年轻代区大小和相应的Survivor区比例 设置此选项后,并行收集器会自动选择年轻代区大小和相应的Survivor区比例,以达到目标系统规定的最低相应时间或者收集频率等,此值建议使用并行收集器时,一直打开.

只需要把基本的内存数据设置好(-Xmx设置最大堆) 使用MaxGCPauseMillis参数(更关注最大停顿时间)或GCTimeRatio参数给虚拟机设立一个优化目标

#### -XX:MaxTenuringThreshold
-XX:MaxTenuringThreshold 垃圾最大年龄
### CMS
#### XX：CMSInitiatingOccupancyFraction
XX：CMSInitiatingOccupancyFraction 使用cms作为垃圾回收 cms的启动阈值 默认92
#### -XX+UseCMSCompactAtFullCollection
-XX+UseCMSCompactAtFullCollection full GC时对老年代的压缩 默认开启 可能会影响性能但减少碎片
#### -XX:CMSFullGCsBeforeCompaction
-XX:CMSFullGCsBeforeCompaction 多少次后进行内存压缩 默认0
