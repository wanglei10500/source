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
应用: Java C#等
![jvm](http://img.blog.csdn.net/20160514180110374)

基本思路：通过”GC Roots”作为起始点开始向下搜索，搜索走过的路径叫做引用链(Reference Chain),当一个对象没有任何引用链时，则是不可用的 判定可回收

可作为GC Roots的对象包括下面几种：
* 虚拟机栈（栈帧中的本地变量表）中引用的对象
* 方法区中类静态属性引用的对象
* 方法区中常量引用的对象
* 本地方法栈中JNI(即Native方法)引用的对象

### 谈引用
引用技术算法和可达性分析算法都与引用有关

引用的传统定义：如果reference类型的数据中存储的数值代表的是另外一块内存的起始地址，就称这块内存代表着一个引用

java1.2之后 引用分为强引用(Strong Reference)、软引用(Soft Reference)、弱引用(Weak Reference)、虚引用(Phantom Reference) 引用强度依次减弱
* 强引用普遍存在的、类似“Object obj=new Object()” 这类的引用，只要强引用还存在。垃圾收集器永远不会回收掉被引用的对象
* 软引用描述还有用但非必须的对象 在系统将要内存溢出前 将会把这些对象列进回收范围之中进行二次回收，如果这次回收还没有足够的内存,才会抛出内存溢出异常。 SoftReference类来实现软引用
* 弱引用描述非必需对象 强度比软引用更弱 被弱引用关联的对象只能生存到下一次垃圾收集发生之前 垃圾收集器工作时 无论当前内存是否足够 都会被回收 WeakReference类来实现弱引用
* 虚引用也称为幽灵引用或幻影引用，为对象设置虚引用的唯一目的是在被收集器回收时收到一个系统通知 PhantomReference类来实现虚引用。

如果没有GC Roots相连接的引用链 至少两次标记

如果没有引用链将会被第一次标记并进行一次筛选 筛选的条件是此对象是否有必要执行finalize()方法 当对象没哟覆盖finalize()方法或者finalize()方法已经被虚拟机调用过 就被视为"没有必要执行"

如果被判定为有必要执行finalize() 对象将被放置在F-Queue队列中，并在之后由一个虚拟机自动创建的、低优先级的Finalizer线程去执行它。 执行指虚拟机会触发不承诺等待运行结束

原因是 如果一个对象finalize() 中执行缓慢或发生死循环 可能导致F-Queue队列中其他对象永久等待，甚至导致整个内存回收崩溃

之后会对F-Queue中的对象进行第二次标记，对象要在finalize中拯救自己，需要重新与引用链的任何一个对象建立关系即可，对象可被移除队列

finalize()方法都只会被系统自动调用一次 但避免主动使用
### 回收方法区
堆中新生代常规一次垃圾回收一般可回收70%-95%

永久代主要回收两部分：废弃常量和无用的类

回收废弃常量与堆中对象类似 常量池中的类(借口)、方法、字段没有其他地方引用，就会被系统清理出常量池

判断无用的类的三个条件：
* 该类所有的实例都已经被回收，也就是Java堆中不存在该类的任何实例。
* 加载该类的ClassLoader已经被回收
* 该类对应的java.lang.Class 对象没有在任何地方被引用，无法在任何地方通过反射访问该类的方法

是否对类进行回收可通过 -Xnoclassgc 如果使用Spring、hibernate这些框架用到很多反射，产生大量临时的class、不能打开

## 垃圾收集算法
### 标记-清除算法(Mark-Sweep)算法
![jvm](http://static.oschina.net/uploads/img/201303/18092408_nT2F.jpg)
在标记完成后统一回收所有被标记的对象

不足：
* 效率问题，标记和清楚两个过程效率都不高
* 空间问题, 标记清楚之后会产生大量不连续的内存碎片 碎片过多会导致之后需要分配较大对象无法找到连续内存不得不提前触发另一次垃圾收集动作
### 复制算法(Copying)
![jvm](http://static.oschina.net/uploads/img/201303/18092409_AaxY.jpg)
可用内存按容量划分为大小相等的两块，每次只使用其中的一块，这一块用完了就将还存活的对象复制到另外一块上面，然后把已使用过得内存空间一次清理掉

代价：将内存缩小为原来的一半

现在商业虚拟机用这种收集算法回收新生代，不需按照1：1分配 可将内存分配为一块较大的Eden空间和两块较小的Survivor空间，每次使用Eden和一快Survivor空间

当回收时，将Eden和Survivor中还存活着的对象一次性地复制到另外一块Survivor空间上 大小为8：1,只有10%机会被浪费

如果每次被回收多于10%存活，当Survivor空间不够用时，需要依赖其他内存(老年代)进行分配担保(Handle Promotion)
### 标记-整理算法(Mark-Compact)
![jvm](http://static.oschina.net/uploads/img/201303/18092410_aV8b.jpg)
复制收集算法在对象存活高的时候进行较多复制操作、效率将变低 若不想浪费50%空间 就需要空间进行分配担保，在老年代一般不能选用这种方法

标记过程与”标记消除算法"相同，后续让所有存活的对象都向一端移动，直接清理掉端边界以外的内存
### 分代收集算法(Generational Collection)
当前商业虚拟机的垃圾收集都采用分代收集算法，根据对象存活周期将内存划分几块，一般把堆分为新生代和老年代

新生代采用复制算法 老年代使用”标记-清理“或”标记-整理“算法

### HotSpot的算法实现

#### 枚举根节点
枚举跟节点 找GC Roots节点必须保证一致性 保证对象引用关系不再变化，这点会导致GC进行时必须停顿Java执行线程

OopMap数据结构存放着对象的引用

#### 安全点
在OopMap的协助下，HotSpot可以快速准确完成GC Roots枚举

问题可能导致引用关系变化、OopMap内容变化的指令非常多，所以只在特定的位置记录这些信息、称为安全点(Safepoint)

程序只有在安全点才能停下来开始GC 方法调用、循环跳转、异常跳转具有这些产生安全点

对于Safepoint，另一个考虑是如何在GC发生时所有线程(不包括执行JNI调用的线程)都跑到安全点再停顿下来 两种方案
* 抢先式中断(Preemptive Suspension) 不需要线程的执行代码去配合，GC发生时，首先把所有线程全部中断，如果发现有线程中断不再安全点上，就恢复线程、让他跑到安全点，几乎没有虚拟机应用这种
* 主动式中断(Voluntary Suspension) 当GC需要中断线程的时候、不直接对线程操作，仅设置标志位，各线程轮询时主动去轮询这个标志，发现中断标志为真就自己中断挂起
#### 安全区域
当程序没有分配CPU时间，线程Sleep或Blocked状态，这时线程无法响应JVM中断请求，走到安全的区域挂起，需要安全区域(Safe Region)解决。

指一段代码片段中，引用关系不会发生变化，任意地方开启GC都是安全的
### 垃圾收集器
![jvm](http://upload-images.jianshu.io/upload_images/650075-8c5080659578032d.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
存在连线即可以搭配使用 所在区域代表是新生代收集器还是老年代收集器
减少工作线程因内存回收而导致停顿Serial收集器->Parallel收集器->Concurrent Mark Sweep(CMS)->GC
#### Serial收集器
![jvm](http://upload-images.jianshu.io/upload_images/650075-ad2db11e4506ceab.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

单线程收集器 它的单线程意义不仅仅说明它只会使用一个CPU或一条收集线程去完成垃圾收集工作，重要的是在它进行垃圾收集时，必须暂停其他所有的工作线程，直到收集结束。Stop The World

虚拟机运行在Client模式下的默认新生代收集器 优点：简单而高效 单个CPU的环境 Serial收集器没有线程交互的开销 可以获得最高的单线程收集效率效率 Client模式虚拟机好的选择

#### ParNew收集器
![jvm](http://upload-images.jianshu.io/upload_images/650075-483c1885e2d36f65.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

Serial收集器多线程版本 Serial收集器可用的所有控制参数(例如:-XX:SurvivorRatio -XX：PretenureSizeThreshold -XX:HandlePromotionFailure等)、收集算法、Stop The World、对象分配规则、回收策略等都与Serial收集器完全一样

运行在Server模式下虚拟机 除了Serial收集器外，只有它能与CMS配合工作

默认开启的收集线程数与CPU的数量相同，在CPU非常多的环境下，可以使用-XX:ParallelGCThreads参数来限制垃圾收集器的线程数

##### 并发与并行
* 并行(Parallel):多条垃圾收集线程并行工作，但此时用户线程仍然处于等待状态。
* 并发(Concurrent):用户线程与垃圾收集线程同时执行(但不一定是并行的，可能会交替执行)，用户程序在继续运行，而垃圾收集程序运行于另一个CPU上

### Parallel Scavenge收集器
使用复制算法 并行多线程收集器

关注点与其他收集器不同 CMS等收集器的关注点是尽可能缩短垃圾收集时用户线程的停顿时间 Parallel Scavenge收集器的目标则是达到一个可控制的吞吐量(Throughput)

吞吐量：CPU用于运行用户代码的时间与CPU总消耗时间的比值， 吞吐量=运行用户代码时间/(运行用户代码时间+垃圾收集时间)

高吞吐量可以高效率的利用CPU时间，适合在后台运算而不需要太多交互的任务

提供两个参数用于精确控制吞吐量：控制最大垃圾收集停顿时间的-XX:MaxGCPauseMillis 直接设置吞吐量大小的-XX:GCTimeRatio

Parallel Scavenge收集器也经常称为 吞吐量优先收集器

GC自适应的调节策略(GC Ergonomics)：Parallel Scavenge收集器参数-XX:+UseAdaptiveSizePolicy 这个参数打开后，就不需要手工指定新生代大小-Xmn、Eden与Survivor区的比例(-XX:Survivor)、晋升老年代对象年龄(-XX:PretenureSizeThreshold)等细节参数
### Serial Old收集器
![jvm](http://upload-images.jianshu.io/upload_images/650075-a877eba6d1753013.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
Serial 收集器的老年代版本 单线程 使用标记-整理算法

应用场景：
* Client模式 Serial Old收集器的主要意义也是在于给Client模式下的虚拟机使用。
* Server模式 Server模式下，两大用途：一种用途是在JDK 1.5以及之前的版本中与Parallel Scavenge收集器搭配使用，另一种用途就是作为CMS收集器的后备预案，在并发收集发生Concurrent Mode Failure时使用。
### Parallel Old收集器
![jvm](http://upload-images.jianshu.io/upload_images/650075-0f74222185d67afc.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
Parallel Scavenge收集器老年代版本 多线程和标记-整理算法

注重吞吐量以及CPU资源敏感的场合 都可以优先考虑Parallel Scavenge+Parallel Old收集器
### CMS收集器(Concurrent Mark Sweep)
![jvm](http://upload-images.jianshu.io/upload_images/650075-b50cffd6ed9e15df.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
以获取最短回收停顿时间为目标的收集器 标记-清除算法 并发收集 低停顿

1. 初始标记(CMS initial mark)
2. 并发标记(CMS Concurrent Mark)
3. 重新标记(CMS remark)
4. 并发清除(CMS Concurrent sweep)

初始标记和重新标记这两个仍然需要”Stop The World“
* 初始标记:仅仅标记GC Roots能直接关联到的对象，速度很快
* 并发标记:就是进行GC Roots Tracing的过程
* 重新标记:修正并发标记期间因用户程序继续运作而导致标记产生变动的那一部分对象的标记记录，停顿时间比初始标记稍长，远比并发标记时间短

缺点：
* 对CPU资源非常敏感 虽然不会导致停顿但会因为占用一部分线程(CPU资源)而导致应用程序变慢 总吞吐量降低 默认启动的回收线程数是(CPU数量+3)/4 随着CPU数量的增加而下降 如果CPU数量不足 会导致过慢
* CMS收集器无法处理浮动垃圾(Floating Garbage) 可能出现"Concurrent Mode Failure" 失败而导致另一次Full GC的产生
  浮动垃圾：由于CMS并发清理阶段用户线程伴随运行 会有新的垃圾产生，这一部分垃圾出现在标记之后、CMS无法在当次收集中处理掉它们 只好留待下一次GC时再清理掉。可使用-XX：CMSInitiatingOccupancyFraction提高触发百分比 设置的太高也会导致大量”Concurrent Mode Failure”
* 标记-清除算法会有大量空间碎片产生 CMS收集器提供一个-XX:+UseCMSCompactAtFullCollection开关参数(默认开启) CMD顶不住开始Full GC时对内存碎片合并整理，非并发
    -XX:CMSFullGCsBeforeCompaction 设置执行多少次不压缩的Full GC后跟着来一次带压缩的(默认0 每次都进行碎片整理)
### G1收集器(Garbage-First)
![jvm](http://upload-images.jianshu.io/upload_images/650075-6c9c9253495eb757.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

服务器端垃圾收集器 特点：
* 并行与并发:G1能充分利用多CPU、多核环境下的硬件优势 使用多个CPU来缩短Stop-The-World停顿时间 部分其他收集器原本需要停顿Java线程执行的GC动作，G1收集器仍然可以通过并发的方式让Java程序继续执行。
* 分代收集:与其他收集器一样，分代概念在G1中依然得以保留。虽然G1可以不需要其他收集器配合就能独立管理整个GC堆，但它能够采用不同的方式去处理新创建的对象和已经存活了一段时间、熬过多次GC的旧对象以获取更好的收集效果。
* 空间整合:与CMS不同 G1从整体看基于标记-整理算法实现 从局部(Region)看是基于复制 这两种在G1运作期间不会产生空间碎片 有利于程序长时间运行
* 可预测的停顿:可让使用者明确指定在一个长度为M毫秒的时间片段内，消耗在垃圾收集上的时间不得超过N毫秒。

G1前收集范围整个新生代或者老年代， G1将整个堆划分为多个大小相等的独立区域(Region) 新生代和老年代不再是物理隔断的，都是一部分Region(不需要连续)的集合

G1跟踪各个Region里面的垃圾堆积的价值大小(回收所获得的空间大小以及回收所需时间的经验值) 在后台维护优先列表，每次根据允许的收集时间 优先回收价值最大的Region(Garbage-First名称的由来) 这种使用Region划分内存空间以及有优先级的区域回收、保证了G1收集器有限时间高效率

对应Region的Remembered Set保证不用全堆扫描

步骤：
* 初始标记(Initial Marking):初始标记阶段只标记GC Roots能直接关联到的对象，并且修改TAMS(Next Top at Mark Start)的值，让下一阶段用户程序并发运行时，能在正确可用的Region中创建新对象，这阶段需要停顿线程，但耗时很短。
* 并发标记(Concurrent Marking):是从GC Root开始对堆中对象进行可达性分析，找出存活对象，耗时较长，可与用户程序并发执行。
* 最终标记(Final Marking):修正在并发标记期间因用户程序继续运作而导致标记产生变动的部分，虚拟机将这段时间对象变化记录在线程Remembered Set Logs里面 最终标记阶段需要把Remembered Set Logs的数据合并到Remembered Set中，需要停顿线程 可并行执行
* 筛选回收(Live Data Counting and Evacuation):首先对各个Region的回收价值和成本排序，根据用户期望的GC停顿时间制定回收计划 可以做到与用户线程并发执行，但只回收一部分，停顿可大幅提高效率

### 理解GC日志
```
[GC (Allocation Failure) [PSYoungGen: 8142K->992K(9216K)] 8142K->5121K(19456K), 0.0078860 secs] [Times: user=0.01 sys=0.02, real=0.01 secs]
[GC (Allocation Failure) --[PSYoungGen: 9184K->9184K(9216K)] 13313K->19416K(19456K), 0.0119785 secs] [Times: user=0.04 sys=0.00, real=0.01 secs]
[Full GC (Ergonomics) [PSYoungGen: 9184K->0K(9216K)] [ParOldGen: 10232K->9875K(10240K)] 19416K->9875K(19456K), [Metaspace: 3065K->3065K(1056768K)], 0.1282468 secs] [Times: user=0.44 sys=0.00, real=0.13 secs]
[Full GC (Ergonomics) [PSYoungGen: 7630K->7912K(9216K)] [ParOldGen: 9875K->7765K(10240K)] 17506K->15678K(19456K), [Metaspace: 3065K->3065K(1056768K)], 0.1140456 secs] [Times: user=0.40 sys=0.01, real=0.11 secs]
[Full GC (Allocation Failure) [PSYoungGen: 7912K->7899K(9216K)] [ParOldGen: 7765K->7765K(10240K)] 15678K->15664K(19456K), [Metaspace: 3065K->3065K(1056768K)], 0.0587146 secs] [Times: user=0.28 sys=0.00, real=0.06 secs]
 ```
* [GC [Full GC 说明这次垃圾收集停顿类型 Full GC 会 Stop The World
* Allocation Failure 引起垃圾回收的类型 本次GC是因为年轻代中没有任何合适的区域能够存放需要分配的数据结构而触发
* 垃圾收集器名 发生区域 DefNew：Default New Generation Serial收集器 | ParNew:Parallel New Generation ParNew收集器 |PSYoungGen Parallel Scavenge
* 方括号内8142K->992K(9216K) 含义是 GC前该区域内存已使用容量->GC后该内存区域已使用容量(该内存区域总容量)
* 方括号外8142K->5121K(19456K) 含义是 GC前Java堆已使用容量->GC后Java堆已使用容量(堆总容量)
* 0.0078860 secs 该内存区域GC所占用的时间
* [Times: user=0.01 sys=0.02, real=0.01 secs] user：用户态消耗的CPU时间 sys:内核态消耗的CPU时间 real:操作从开始到结束所经过的墙钟时间(Wall Clock Time)
    墙钟时间包含各种非运算的等待耗时，例如等待磁盘IO，等待线程阻塞 多CPU会叠加CPU时间 所以user、sys可能比real多

## 内存分配与回收的策略

### 对象优先在Eden分配
大多数情况下 对象在Eden区中分配 Eden区没有足够空间分配时 jvm会发起一次Minor GC

-Xms20M -Xmx20M -Xmn10M 限制了Java堆大小为20MB 不可扩展 其中10MB分配给新生代 剩下10MB分配给老年代

* 新生代GC(Minor GC):发生在新生代的垃圾收集动作 频繁 回收速度较快
* 老年代GC(Majar GC/Full GC):发生在老年代GC 经常伴随至少一次MinorGC 速度一般比Minor GC慢10倍以上
### 大对象直接进入老年代
大对象：需要大量连续内存空间的Java对象 很长的字符串以及数组 经常出现大对象很容易导致内存还有不少空间时就提前触发垃圾收集以获取足够的连续空间来安置它们

-XX：PretenureSizeThreshold参数 大于这个设置值的对象直接在老年代分配，避免Eden区及两个Survivor区之间发生大量的内存复制
### 长期存活的对象将进入老年代
如果对象在Eden出生并经过第一次Minor GC后仍然存活，并且能被Survivor容纳的话，将被移动到Survivor空间中，并且对象年龄设为1.

对象在Survivor区中每熬过一次Minor GC,年龄就增加1岁，当增加到一定程度(默认15)被晋升到老年代中。阈值通过-XX:MaxTenuringThreshold设置
### 动态对象年龄判定
如果在Survivor空间中相同年龄所有对象大小哦总和大于Survivor空间的一半 年龄大于或等于该年龄的对象就可以直接进入老年代
### 空间分配担保
Minor GC之前，虚拟机会先检查老年代最大可用的连续空间是否大于新生代所有对象总空间 如果成立 MinorGC确保安全 如果不成立

jvm查看HandlePromotionFailure设置值是否允许担保失败 如果允许 继续检查老年代最大可用的连续空间是否大于历次晋升到老年代对象的平均大小 如果大于 尝试MinorGC 如果小于或不允许担保失败 改为Full GC

jdk6 Update24之后规则变为只要老年代的连续空间

## 虚拟机性能监控与故障处理工具

### jps: 虚拟机进程状况工具(JVM Process Status Tool)
列出正在运行的虚拟机进程 显示虚拟机执行主类(Main Class) 名称以及这些进程的本地虚拟机唯一ID(LVMID) LVMID与PID是一致的

jps命令格式：

jps [options] [hostid]

jps执行样例：
```
jps -l

5892 org.jetbrains.idea.maven.server.RemoteMavenServer
28998 sun.tools.jps.Jps
5790 com.intellij.idea.Main
```
选项:
* -l 输出主类的全名，如果进程执行的是Jar包，输出Jar路径
* -v 输出虚拟机进程启动时的JVM参数
* -m 输出虚拟机进程启动时传递给主类Main()的参数

### jstat:虚拟机统计信息监视工具(JVM Statistics Monitoring Tool)
监视虚拟机各种运行状态信息的命令行工具

jstat命令格式为：

jstat [option vmid [interval[s|ms] [count]]]

interval与count代表查询间隔和次数
```
jstat -gc 2764 250 20  每250毫秒查询一次进程2764垃圾收集状况 一共查询20次
```
demo
```
jstat -gcutil 2764
结果： 这台服务器的新生代Eden区(E,表示Eden)
      两个Survivor区(S0 S1 表示Survivor0\Survivor1)
      老年代(O,表示Old)
      永久代(P,表示Permanent)
      Minor GC(YGC 表示young GC)
      Full GC(FGC 表示Full GC)
      Full GC总耗时(FGCT 表示Full GC Time)
      所有GC总耗时(GCT 表示GC Time)
```
### jinfo: Java配置信息工具(configuration Info for Java)
实时查看和调整虚拟机各项参数

jinfo [option] PID

样例：查询CMSInitiatingOccupancyFraction参数值
```
jinfo -flag CMSInitiatingOccupancyFraction 1444
```
### jmap: Java内存映像工具(Memory Map for Java)
用于生成堆转储快照 获取dump文件 查询finalize执行队列、Java堆永久代的详细信息 如空间使用率、当前用的哪种收集器

jmap命令格式:

jmap [option] vmid
