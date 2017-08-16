---
title: jvm
tags:
 - jvm
 - java
categories: 经验分享
---
![jvm](http://upload-images.jianshu.io/upload_images/2614605-246286b040ad10c1.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## 运行时数据区域
有的随着虚拟机进程的启动而存在，有些区域依赖用户线程的启动和结束而建立和销毁
### 程序计数器(Program Counter Register) PC
较小的内存空间、当前线程所执行的字节码的行号指示器。字节码解释器工作时通过改变这个计数器的值选取下一条需要执行的字节码指令。

线程级 随线程的创建而产生、随线程的销毁而销毁

### Java虚拟机栈(Java Virtual Machine Stacks)
线程私有 描述的是Java方法执行的内存模型：每个方法执行的同时会创建一个栈帧(Stack Frame)用于存储局部变量表、操作数栈、动态链接、方法出口等信息。 每个方法调用到执行完成对应栈帧在虚拟机栈由入栈到出栈

栈帧主要包括：局部变量表和操作数栈

一般的堆内存、栈内存的划分中的栈指的是虚拟机栈中局部变量表部分

局部变量表存放了编译期可知的基本数据类型(boolean byte char short int float long double)、对象引用(reference类型)、returnAddress类型(指向了一条字节码指令的地址)

这个区域异常：
* StackOverflowError: 线程请求的栈深度大于虚拟机允许的深度
* OutOfMemoryError:如果虚拟机栈可以动态扩展，扩展时也无法申请到足够的内存

### 本地方法栈(Native Method Stack)
与JVM栈基本相同，可能合二为一，针对Native方法。也有StackOverflowError和OutOfMemoryError异常

###Java堆(Java Heap)
虚拟机管理线程最大的一块 被所有线程共享的一块内存区域 虚拟机启动而创建 唯一目的是存放对象实例

所有的对象实例以及数组都要在堆上分配(不绝对)

Java堆是垃圾收集器管理的主要区域，也叫“GC堆”

因为收集器大多使用分代收集 Java堆可细分为新生代和老年代，再细致有Eden空间、FromSurvivor空间、To Survivor空间等

Java堆可以处理物理上不连续的内存空间、主流的虚拟机为可扩展的(通过-Xmx -Xms控制)

这个区域异常：
* OutOfMemoryError：堆内没有内存完成实例分配，并且也无法再扩展
### 方法区(Method Area)
线程共享 用于存储已被虚拟机加载的类信息、常量、静态变量、即时编译器编译后的代码等数据

这个区域的内容很难被回收 永久代形容方法区 有个特殊的区域被划分(逻辑)出来，即运行时常量池

这个区域的回收针对常量池的回收和对类型的卸载

这个区域的异常：
* OutOfMemoryError:方法区无法满足内存分配需求时
#### 运行时常量池(Runtime Constant Pool)
是方法区的一部分
