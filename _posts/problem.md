---
title:
tags:
 -
 -
categories: 经验分享
---


二进制
```
1byte(B)=8bits(b)字节=8个二进制位 (字节)
1(K/KB)=2^10bytes=1024bytes个字节
1(M/MB)=2^20bytes=1048576bytes个字节
1(G/GB)=2^30bytes=~
1(T/TB)=2^40bytes=~
```
十进制
```
1bytes(B)=8bits(b)
1K/KB=10^3bytes=1000 bytes
1M/MB=10^6bytes=100 000 bytes
1G/GB=10^9bytes=100 000 000bytes
1T/TB=10^12bytes=100 000 000 000bytes
```
```
1GB=1024MB
   =1024*1024KB
   =104*1024*1024byte

```
给一个超过100G大小的log file, log中存着IP地址, 设计算法找到出现次数最多的IP地址？
与上题条件相同，如何找到top K的IP？如何直接用Linux系统命令实现？
链接：https://www.nowcoder.com/questionTerminal/4879439a1e444e8ebff2232d206e05e7

Hash分桶法：
• 将100G文件分成1000份，将每个IP地址映射到相应文件中：file_id = hash(ip) % 1000
• 在每个文件中分别求出最高频的IP，再合并 Hash分桶法：
• 使用Hash分桶法把数据分发到不同文件
 • 各个文件分别统计top K
• 最后Top K汇总
Linux命令，假设top 10：sort log_file | uniq -c | sort -nr k1,1 | head -10
cat ip.txt| awk '{print $1}' | sort | uniq -c | sort -nr | head -10



分析：
  由于文件超过100G，所以我们必须先对文件进行切分，然后再利用数据结构的知识求解。关键是如何切分效率最高？？？

解决方法：
  我们可以使用哈希切分，将同一个ip都分割到同一个文件中，注意同一个ip经过同一个散列函数 一定会进入到同一个文件中，然后再统计每一个文件中出现最多的ip的次数，最后再将这些结果放到一起再经过比较就能得出结果。

问题：
  最极端的情况是经过哈希切分后有一个ip出现次数特别多，切分后存放这个ip的文件还是太大加载不到内存中，这时候应该怎么办呢？？？
  这时候可以将这个文件再切分(普通的切分)成若干个小文件，将每一个文件加载到内存中使用hash表求出ip出现的次数，最后再将这些小文件的结果汇总后在求出出现次数最多的ip。


2、与上题条件相同，如何找到top K的ip?
  与第1题一样，我们统计出每个文件中每个ip出现的次数然后再利用最小堆分别求出每个文件中出现次数最多的前k个ip，最后再将这些文件的结果汇总到一起再利用最小堆求出出现次数最多的前K个ip。

  给一个超过100G大小的log file,log中存在着IP地址，设计算法找到出现此数最多的IP地址

1.这里首先想到的是切分，因为呢，100G太大，放不到内存中；
我们可以切分成1000个小文件，每个文件大概就是500M左右的样子；
2.可是，问题来了，如何才可以将相同的IP地址放入同一个文件当中呢？
3.这里就要用到哈希的思想，哈希切分；
每一个IP地址就是一个对应的字符串，我们可以算出key值，然后取余数，余数就表示的是存储的文件号
比如index = key% 1000;余数必定在 0~999，这样我们就可以将相同的IP地址放入同一个文件中啦
然后呢，依次将这些文件装载入内存，统计每个文件出现最多的IP，然后进行比较
即可求解
题目2:
给一个超过100G大小的log file,log中存在着IP地址，设计算法找到TopK的IP地址

1.说到TopK问题，立马想到了堆这个数据结构；
题目中求得是次数最多的前K个，那我们就可以建一个小堆；
2.还是利用到第一道题目，当我们得到第一个文件时，将第一个文件中前K个数据建小堆；
3.然后后面的元素依次和堆顶元素进行比较，如果比堆顶元素的次数多，那么堆顶出堆，该元素入堆
直到所有文件全部读完，留在该堆里面的元素便是TopK个最多的元素




PriorityQueue 最小堆解决TopK问题
最小堆（小根堆）是一种数据结构，它首先是一颗完全二叉树，并且，它所有父节点的值小于或等于两个子节点的值。

最小堆的存储结构（物理结构）实际上是一个数组。
堆有几个重要操作：

BuildHeap：将普通数组转换成堆，转换完成后，数组就符合堆的特性：所有父节点的值小于或等于两个子节点的值。

Heapify(int i)：当元素i的左右子树都是小根堆时，通过Heapify让i元素下降到适当的位置，以符合堆的性质。

TopK问题上，用最小堆的解决方法就是：先取源数据中的K个元素放到一个长度为K的数组中去，再把数组转换成最小堆。再依次取源数据中的K个之后的数据和堆的根节点（数组的第一个元素）比较，根据最小堆的性质，根节点一定是堆中最小的元素，如果小于它，则直接pass，大于的话，就替换掉跟元素，并对根元素进行Heapify，直到源数据遍历结束。
