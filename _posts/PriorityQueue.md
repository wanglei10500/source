
![spark](http://obqtqeg7n.bkt.clouddn.com/java-collection-queue/1.png!zip)

Queue实现 Queue是一种先进先出(FIFO)数据结构 (最大堆，最小堆)

java  PriorityQueue since1.5 是基于优先堆的无界队列实现。优先顺序基于元素实现的Comparable接口(未实现会抛出ClassCastExeption)或者构造函数中传入Comparator比较器。队列的头是优先级最高的元素 不接受null值，非线程安全

如果不提供Comparator的话，优先队列中元素默认按自然顺序排列，也就是数字默认是小的在队列头，字符串则按字典序排列

PriorityQueue的构造方法
* PriorityQueue()使用默认初始容量[11]创建一个PriorityQueue，根据其自然顺序排序其元素(使用Comparable)
* PriorityQueue(int initialCapacity)使用指定的初始容量创建一个PriorityQueue 当达到容量时 PriorityQueue会自动扩容,用size()方法获取队列大小时，返回的是PriorityQueue中真实元素的个数，非initialCapacity
* PriorityQueue(int initialCapacity,Comparator Comparator)使用指定的初始化容量创建一个PriorityQueue，根据指定的比较器Comparator来排序其元素。默认最小堆(e1-e2)
```
构造最大堆
PriorityQueue<Integer> maxHeap=new PriorityQueue<Integer>(10, new Comparator<Integer>() {
            public int compare(Integer o1, Integer o2) {
                return o2-o1;
            }
        });
```
相关性质(最小堆为例)
* 多个最小值问题。 此队列头是最小元素，若多个最小值，头是其中任意一个
* 容量问题。 队列检索操作poll、remove、peek访问处于队列头的元素。优先级队列无界，但有内部容量，控制着用于存储队列元素的数组的大小
* 实现原理。 数组实现，大小可以动态增加，容量无限
* 线程安全问题。 不是同步的，不是线程安全的， 如果多个线程中的任意线程从结构上修改了列表，则这些线程不应同时访问PriorityQueue实例，这时使用PriorityBlockingQueue类
* 元素限制。 不允许使用null元素
* 时间复杂度。 堆(PriorityQueue实际上是用数组表示的二叉完全数) 插入方法(remove() poll() offer(E e) add(E e)) O(log(n))
            remove(Object o)和contains(Object o)方法提供线性时间。为检索方法(peek() element()和size()提供O(1))时间
            解释：poll()操作删除树根元素后，需要进行下滤操作，复杂度O(log(n)) 对于remove(Object o)操作，需要遍历数组
* iterator()方法。 方法提供的迭代器并不保证以有序的方式遍历优先级队列中的元素。 数组存储结构a[0]=1,a[1]=3，a[2]=2,a[3]=5,a[4]=4 如果要按顺序遍历，考虑使用Arrays.sort(pq.toArray()).
* PriorityQueue的内部实现。此类及其迭代器实现了Collection和Iterator接口的所有可选方法。PriorityQueue对元素采用堆排序

peek()方法 检索头部 不移除

poll()方法 检索并移除头部元素

### 应用
* topK
如果求top k大的问题，就建立小根堆，否则大根堆，使队列容量保持在k。  简单方法可以直接使用内置的TreeMap或者TreeSet，这两者是基于红黑树的一种数据结构，内部维持key的次序，每次添加新元素，其排序的开销要大于堆调整哦开销，
而且无法存重复元素。
```
public int findKthLargest(int[] nums, int k){
      if(nums.length==1){
          return nums[0];
      }
      PriorityQueue<Integer> maxHeap = new PriorityQueue(k);
      maxHeap.add(nums[0]);
      for(int i=1;i<nums.length;i++){
          if(maxHeap.size()<k){
              maxHeap.add(nums[i]);
          }else{
              int value = maxHeap.peek();
              if(nums[i]>value){
                  maxHeap.poll();
                 maxHeap.add(nums[i]);
              }
          }
      }
```
* 求中位数
建立一个最大堆和一个最小堆，使两堆的容量差1以下，这个中位数就可以通过两队列 队顶元素求得。
### PriorityQueue在hadoop中的应用：
hadoop中，排序是MapReduce的灵魂，MapTask和ReduceTask均会对数据按key排序，这个操作是MR框架的默认行为，MapReduce框架中，用到的排序主要有两种：快速排序和基于堆实现的优先级队列。

* Mapper阶段：
从map输出到环形缓冲区的数据会被排序(这是MR框架中改良的快速排序)，这个排序涉及partition和key，当缓冲区容量占用80%，会spill数据到磁盘，生成IFile文件，Map结束后，会将IFile文件排序合并成一个大文件(基于堆实现的优先级队列)，以供不同的reduce来拉取相应的数据
* Reducer阶段：
从Mapper端取回的数据已是部分有序，Reduce Task只需进行一次归并排序即可保证数据整体有序。为了提高效率，Hadoop将sort阶段和reduce阶段并行化，在sort阶段，Reduce Task为内存和磁盘中的文件建立了小顶堆，保存了指向该小顶堆根节点的迭代器，
并不断的移动迭代器，以将key相同的数据顺次交给reduce()函数处理，期间移动迭代器的过程实际上就是不断调整小顶堆的过程(建堆-取堆顶元素-重新建堆-取堆顶元素) 这样，sort和reduce可以并行进行
