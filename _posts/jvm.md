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
是方法区的一部分 class文件中 常量池 存放编译期生成的各种字面量和符号引用
这个区域异常：
* OutOfMemoryError:常量池无法再申请到内存时


### 直接内存(Direct Memory)
不是虚拟机运行时数据区的一部分也可能导致OutOfMemoryError异常

1.4后加入的NIO(New Input/Output)类，引入了基于通道(Channel)与缓冲区(Buffer)的IO方式，可以使用Native函数库直接分配对外内存，然后通过Java堆中的DirectByteBuffer对象作为这块内存的引用进行操作，可显著提高性能，避免在Java堆与Native堆中复制数据
这个区域异常：
* OutOfMemoryError：各个内存区域总和大于物理内存限制(包括物理和操作系统的限制)

### 对象的创建+demo
```
public void test (int result, int num) {
    TestClassB classB = new TestClassB();
    classB.methodB(result,num);
}

public class TestClassB {
    public void methodB(result, num) {
        int finalResutl = result + num;
        ......
    }
}

```
假设A在执行test方法，并已经执行到TestClassB classB=new TestClassB()

首先判断类TestClassB有没有被加载到方法区中，如果没有，先加载类入方法区

然后执行new操作，创建一个对象，需要在堆上申请内存，用于存放对象的相关数据(对象头、实例数据、类型指针、占位符等)

为TestClassB的成员赋零值，最后设置对象头。这样对于JVM对象就创建成功了

pc+1 执行下一步 classB.methodB() 这是一个方法调用，方法的执行和结束意味着方法栈中栈帧的进栈和出栈

- 如何划分可用空间？

内存空间分配对象时，空间归整-指针碰撞(Bump the Pointer) 空间不规整-空闲列表(Free List)

选择哪种分配方式由Java堆是否归整决定，由所采用的垃圾收集器是否带有压缩整理功能决定

* 使用Serial、ParNew等带Compact过程的收集器时，系统采用的分配算法是指针碰撞
* 使用CMS这种基于Mark-Sweep算法的收集器时，采用空闲列表。

频繁的分配对象不是线程安全的 解决方案两种
1. 对分配内存空间的动作进行同步处理--虚拟机采用CAS配上失败重试的方式保证更新操作的原子性
2. 把内存分配的动作按照线程划分在不同的空间中进行，即每个线程在堆中预先分配一小块内存，成为本地线程分配缓冲(Thread Local Allocation Buffer,TLAB)只有TLAB用完并分配新的TLAB时，才需要同步锁定。是否使用TLAB可通过-XX:+/-UseTLAB 参数配置

内存分配完成后需要将分配到的内存空间初始化为零值，保证对象实例在Java中不赋初值就可以使用

接下来，虚拟机要对对象进行必要设置，例如这个对象是哪个类的实例、如何才能找到类的元数据信息、对象的哈希码、对象的GC分代年龄等信息。存放在对象头中。

### 对象的内存布局
![jvm](http://upload-images.jianshu.io/upload_images/5401975-4c082ac80e1c042c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
对象在内存中分三块区域:对象头(Header)、实例数据(Instance Data)、对齐填充(Padding)

#### 对象头
* Mark Word 包含一系列的标记位 哈希码、GC分代年龄、锁状态标志、线程持有的锁、偏向线程ID、偏向时间戳。在32位系统占4字节，在64位系统中占8字节
* Class Pointer 用来指向对象对应的Class对象（其对应的元数据对象）的内存地址。虚拟机通过这个指针确定这个对象是哪个类的实例。在32位系统占4字节，在64位系统中占8字节；
* Length：如果是数组对象，还有一个保存数组长度的空间，占4个字节；

#### 实例数据
对象实际数据包括了对象的所有成员变量，其大小由各个成员变量的大小决定，比如：byte和boolean是1个字节，short和char是2个字节，int和float是4个字节，long和double是8个字节，reference是4个字节（64位系统中是8个字节）。

从父类继承、子类中定义的都需要记录下来

#### 对齐填充
不是必然存在 启占位符作用

Java对象占用空间是8字节对齐的，即所有Java对象占用bytes数必须是8的倍数。例如，一个包含两个属性的对象：int和byte，这个对象需要占用8+4+1=13个字节，这时就需要加上大小为3字节的padding进行8字节对齐，最终占用大小为16个字节。

注意：以上对64位操作系统的描述是未开启指针压缩的情况

### 对象的访问定位
java程序通过操作栈上的reference来访问堆上的具体对象
![jvm](http://img.blog.csdn.net/20160808125925120)

* 句柄访问，Java堆中将会划分出一块内存来作为句柄池，reference中存储的就是对象的句柄地址，句柄中包含了对象实例数据与类型数据各自的具体地址信息
![jvm](http://img.blog.csdn.net/20160808125939374)
* 通过直接指针访问,Java堆对象的布局中就必须考虑如何放置访问类型数据的相关信息，而reference中存储的直接就是对象地址

句柄访问好处是reference中存储的是稳定的句柄地址，在对象被移动（垃圾回收）只会改变句柄中的实例数据指针，而reference本身不需要修改

直接指针访问好处是速度更快，节省了一次指针定位时间开销

### OOM异常
分析工具http://www.eclipse.org/mat/
#### Java 堆内存
Java堆用于存储对象实例，只要不断创建对象，并且保证GC Roots到对象之间有可达路径来避免垃圾回收清楚对象。那么达到最大堆容量限制之后就会产生内存溢出异常

参数-XX：+HeapDumpOnOutOfMemoryError可以出现内存溢出时储存快照

使用内存映像分析工具(Eclipse Memory Analyzer) 确认内存泄露(Memory Leak)或内存溢出(Memory Leak)
#### 虚拟机栈和本地方法栈溢出
栈容量由-Xss参数设置

* 如果线程请求的栈深度大于虚拟机所允许的最大深度，将抛出StackOverflowError异常
* 如果虚拟机在扩展栈时无法申请到足够的内存空间，则抛出OutOfMemoryError异常
两种异常很难界定
* 使用-Xss参数减少栈内存容量。结果：抛出StackOverflowError异常，异常出现时输出的堆栈深度相应缩小
* 定义了大量的本地变量，增大此方法帧中本地变量表的长度。结果：抛出StackOverflowError，异常输出的堆栈深度相应缩小

死循环调用函数或创建线程可将虚拟机栈和本地方法栈消耗完 若是线程过多引发OOM可考虑减少线程数或减少最大堆和减少栈容量换取更多的线程
#### 方法区和运行时常量池溢出
String.intern()是Native方法，作用是：如果字符串常量池中已经包含一个等于此String对象的字符串，则返回代表池中这个字符串的String对象;否则 将此string对象包含的字符串添加到常量池中，返回此String对象的引用

1.6前字符串常量池分配在永久代，可通过-XX：PermSize和-XX：MaxPermSize限制方法区大小，从而限制常量池容量

大量的类也会造成异常
#### 本机直接内存溢出
DirectMemory容量可通过 -XX:MaxDirectMemorySize指定 默认为Java堆最大值(-Xmx指定)一样 这部分内存不受JVM垃圾回收管理

DirectBuffer并没有真正向OS申请分配内存，其最终还是通过Unsafe的allocateMemory()来进行内存分配

## 垃圾收集器与内存分配策略

垃圾收集(Garbage Collection,GC)

程序计数器、虚拟机栈、本地方法栈随线程生灭，所需内存基本固定 不需要过多考虑回收的问题

### 引用计数算法
给对象中添加引用计数器，每当有一个地方引用它时 计数器值+1,引用失效时、计数器值-1

应用：COM技术、Python、FlashPlayer Java虚拟机未应用原因：很难解决对象间相互循环引用的问题
### 可达性分析算法
