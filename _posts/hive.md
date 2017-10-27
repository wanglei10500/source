---
title: hadoop
tags:
 - hadoop
 - hive
categories: 经验分享
---
### 数据倾斜原因

####  解决办法
* 好的模型设计
* 解决数据倾斜问题
* 减少job数
* 设置合理的map reduce的task数，能有效提升性能。(比如，10w+级别的计算，用160个reduce 很浪费，1个足够)
* 了解数据分布，set hive.groupby.skewindata=true;这是通用的算法优化，但算法优化有时不能适应特定业务背景。
* 数据量较大的情况下，慎用count(distinct)，count(distinct)容易产生倾斜问题。
* 对小文件进行合并
* 优化时把握整体，单个作业最优不如整体最优。
#### 底层原因
hive性能优化时，把hiveql当作M/R程序来读，即从M/R的运行角度来考虑优化性能，RAC(Real Application Cluster)真正应用集群 Hadoop如果每次只做小数量的输入输出，利用率将会很低，用好hadoop的首要任务是增大每次任务所搭载的数据量

Hadoop的核心能力是partition和sort，因而这也是优化的根本。Hadoop处理数据的特征：
* 数据的大规模并不是负载重点，造成运行压力过大是因为运行数据的倾斜
* jobs数比较多的作业运行效率相对比较低，比如即使有几百张的表，如果多次关联多次汇总，产生十几个jobs，耗时很长。原因是map reduce作业初始化的时间是比较长的。
* sum，count，max，min等UDAF，不怕数据倾斜问题，hadoop在map端的汇总合并优化，使数据倾斜不成问题。
* count(distinct)，在数据量大的情况下，效率较低，如果多count(distinct )效率更低，因为count(distinct)是按group by字段分组，按distinct字段排序，一般这种分布方式很倾斜
  例：男nv，女uv 一天30亿pv，按性别分组，分配两个reduce，每个reduce处理15亿数据
* 数据倾斜是导致效率大幅降低的主要原因，可以采用多一次Map/Reduce的方法，避免倾斜
结论： 避实就虚，用job数的增加，输入量的增加，占用更多存储空间，充分利用空闲CPU等各种方法，分解数据倾斜造成的负担
#### 具体原因
1. key分布不均匀
2. 业务数据本身的特性
3. 建表时考虑不周
4. 某些SQL语句本身就有数据倾斜
#### 具体表现
任务进度长时间维持在99%(或100%)，查看任务监控页面，发现只有少量(1个或几个)reduce子任务未完成，因为其数据量和其他reduce差异过大

单一reduce的记录数与平均记录数差异过大，通常可能达到3倍甚至更多。最长时长远大于平均时长
### 参数调节
#### 列裁剪
Hive在读数据的时候，可以只读取查询中需要的列，忽略其他列
```sql
select a,b from q where e
```
在实施此查询中，q表中有5列(a,b,c,d,e)，hive只读取查询逻辑中真实需要的a,b,e 而忽略c,d 节省读取开销，中间表存储开销和数据整合开销，裁剪对应的参数项为：hive.optimize.cp=true(默认为真)
#### 分区裁剪
可以在查询的过程中减少不必要的分区。例如：
```sql
select * from (select a1,count(1) from t group by a1)subq where subq.prtn=100; #多余分区
select * from t1 join (select * from t2) subq on(t1.a1=subq.a2) where subq.prtn=100;
```
查询语句若将subq.prtn=100条件放入子查询中更为高效，可以减少读入的分区数目。hive自动执行这种裁剪优化。
分区参数为：hive.optimize.pruner=true(默认值为真)
#### 笛卡尔积
当hive设定为严格模式(hive.mapred.mode=strict)时，不允许在HQL语句中出现笛卡尔积。

当无法躲避笛卡尔积时，采用MapJoin,会在Map端完成Join操作，将Join操作的一个或多个表完全读入内存。MapJoin的用法是在查询/子查询的select关键字后面添加/+MAPJOIN(tablellist)/ 提示优化起转化为MapJoin。其中tablelist可以是一个表，或以逗号连接的表的列表。
tablelist中的表会读入内存，应该将小表写在这里
#### join操作
在编写带有join操作的代码语句时，应该将条目少的表/子查询放在join操作符的左边。因为在Reduce阶段，位于Join操作符左边的表的内容会被加载进内存。载入条目较少的表可以有效减少OOM(out of memory)即内存溢出。

所以对于同一个key来说，对应的value值小的放前，大的放后，这便是”小表放前“原则。若一条语句中有多个Join，依据Join条件相同与否，有不同的处理方法。
##### join原则
对于一条语句中有多个Join的情况，如果Join的条件相同，比如查询：
```sql
insert overwrite table pv_users select pv.pageid,u.age from page_view p join user u on(pv.userid=u.userid) join newuser x on(u.userid=x.userid);
```
* 如果Join的key相同，不管有多少个表，都会合并为一个Map-Reduce
* 一个Map-Reduce任务，而不是n个
* 在做OUTER JOIN的时候也是一样
* 如果Join的条件不相同，Map-Reduce的任务数目和Join操作的数目是对应的，以下查询是等价的 比如：
```sql
insert overwrite table pv_users select pv.pageid,u.age from page_view p join user u on(pv.userid=u.userid) join newuser x on(u.age=x.age);

insert overwrite table tmptable select * from page_view p join user u on(pv.userid=u.userid);

insert overwrite table pv_users select x.pageid,x.age from tmptable x join newuser y on (x.age=y.age);   
```
##### Map Join
Join操作在Map阶段完成，不再需要Reduce，前提条件是需要的数据在Map的过程中可以访问到。比如查询：
```sql
insert overwrite table pv_users select /*+ MAPJOIN(pv) */ pv.pageid,u.age from page_view pv join user u on (pv.userid=u.userid);
```
![spark](https://ych0112xzz.github.io/2017/02/13/Hive-Skew/mapjoin.png)
相关的参数为：
* hive.join.emit.interval=1000
* hive.mapjoin.size.key=10000
* hive.mapjoin.cache.numrows=10000
##### 空值产生的数据倾斜
* 场景：如日志中，常会有信息丢失的问题，比如日志中的user_id，如果取其中的user_id和用户表中的user_id关联，会碰到数据倾斜的问题
* 解决办法1：为空的不参与关联
```sql
select * from log a
  join users b
  on a.user_id is not null
  and a.user_id=b.user_id
union all
  select * from log a
  where a.user_id is null;
```
* 解决办法2：赋予空值新的key
```sql
select * from log a left outer join users b on case when a.user_id is null then concat('hive',rand()) else a.user_id end = b.user_id;
```
* 结论：方法2比方法1效率更好，不但io少了，而且作业数也少了。解决方法1中log读取两次，jobs是2。 解决方法2job数是1.这个优化适合无效id(比如-99,‘’，null等)产生的倾斜问题。把空值的key变成一个字符串加上随机数，就能把倾斜的数据分到不同的reduce上，解决数据倾斜问题。
##### 不同数据类型关联产生数据倾斜
* 场景：用户表中user_id字段为int，log表中user_id字段既有string类型也有int类型。 当按照user_id进行两个表的Join操作时，默认的Hash操作会按int型的id来进行分配，这样会导致所有string类型id的记录都分配到一个reduce中
* 解决办法：把数字类型转换成字符串类型
```sql
select * from users a left outer join logs b on a.user_id = cast(b.user_id as string);
```
##### 小表不小不大，怎么用map join解决倾斜问题
使用map join解决小表(记录数少) 关联大表的数据倾斜问题，这个方法使用的频率非常高，但如果小表很大，大到map join会出现异常，这时就需要特别的处理
```sql
select * from log a left outer join users b on a.user_id=b.user_id;
```
users表有600w+的记录，把users分发到所有的map上也是个不小的开销，而且map join不支持这么大的小表。如果用普通的join 又会碰到数据倾斜的问题
* 解决方法：
```sql
select /*+mapjoin(X)*/ * from log a
left outer join (
  select /*+mapjoin(c)*/ d.*
  from (select distinct user_id from log) c
  join users d
  on c.user_id=d.user_id
)x
on a.user_id=b.user_id;
```
假如，log里user_id有上百万个，就回到了原来的map join问题。所幸，每日会员uv不会太多，有操作的不会太多。。。
### LEFT SEMI JOIN
是IN/EXISTS子查询的一种更高效的实现。

left semi join 的限制是，join子句中右边的表只能在on子句中设置过滤条件
```sql
select a.key,a.value from a where a.key in (select b.key from b);   ->

select a.key,a.value from a left semi join b on (a.key=b.key);
```
### group by
进行group by操作时需要注意一下几点：
* Map端部分聚合
事实上并不是所有的聚合操作都需要在reduce部分进行，很多聚合操作都可以先在Map端进行部分聚合，然后reduce端得出最终结果

这里需要修改的参数为：
hive.map.aggr=true(用于设定是否在map端进行聚合，默认值为真，相当于combine) hive.groupby,mapaggr.checkinterval=100000(用于设定map端进行聚合操作的条目数)
* 有数据倾斜时进行负载均衡
此处需要设定hive.groupby.shewindata。当选项设定为true是，生成的查询计划有两个MapReduce任务。

第一个MapReduce中，map的输出结果集合会随机分布到reduce中，每个reduce做部分聚合操作，并输出结果。这样处理的结果是，相同的group by key有可能分发到不同的reduce中，从而达到负载均衡的目的。

第二个MapReduce任务再根据预处理的数据结果按照group by key分布到reduce中(这个过程可以保证相同的group by key分布到同一个reduce中)，最后完成最终的聚合操作
### 控制Map数和Reduce数
#### 控制Map数
同时可执行的map数是有限的。
* 通常情况下，作业会通过input的目录产生一个或者多个map任务
* 主要的决定因素有：input的文件总个数，input的文件大小。
* 举例
1. 假设input目录下有一个文件a,大小为780M,那么hadoop会将该文件a分隔成7个块(block为128M,6个128M的块和1个12m的块)，从而产生7个map数
2. 假设input目录下有3个文件a,b,c大小分别为10m，20m，130m，那么hadoop会分隔成4个块(10m,20m,128m,2m)，从而产生4个map数
* 两种方式控制Map数
1. 减少map数可以通过合并小文件来实现，这点是对文件数据源来讲
2. 增加map数的可以通过控制上一个job的reduce数来控制
#### 控制reduce数
* reducer个数的设定极大影响执行效率不指定reducer个数的情况下，Hive分配reducer个数是基于以下：
1. 参数1：hive.exec.reducers.bytes.per.reducer(默认为1G)
2. 参数2：hive.exec.reducers.max(默认为999)
* 计算reducer数的公式
1. N=min(参数2,总数据量/参数1)
2. set mapred.reduce.tasks=13;
* reduce个数并不是越多越好。同map一样，启动和初始化reduce也会消耗时间和资源，有多少个reduce，就会有多少个输出文件
1. Reducer数过多：生成了很多个小文件，那么如果这些小文件作为下一个任务的输入，则也会出现小文件过多的问题。
2. Reducer过少：影响执行效率。
* 什么情况下只有一个reduce
很多时候会发现不管数据量多大，不管有没有设置调整reduce个数的参数，任务中一直都只有一个reduce任务;
* 除了数据量小于hive.exec.reducers.bytes.per.reducer参数值的情况外
* 没有group by 的汇总
* 用了order by
### 合并MapReduce操作
* Multi-group by :当从同一个源表进行多次查询时用。Multi-group by是Hive的一个非常好的特性，它使得Hive中利用中间结果变得非常方便
```sql
from log
insert overwrite table test1 select log.id group by log.id
insert overwrite table test2 select log.name group by log.name  
```
上述查询语句使用了Multi-group by特性连续group by了2次数据，使用不同的group by key 这一特性可以减少一次MapReduce操作
### 合并小文件
文件数目小，容易在文件存储端造成瓶颈，给HDFS带来压力，影响处理效率。对此，可以通过合并Map和Reduce的结果文件来消除这样的影响。

用于设置合并属性的参数有：
* 是否合并Map输出文件：hive.merge.mapfiles=true(默认值为真)
* 是否合并Reduce端输出文件：hive.merge.mapredfiles=false(默认值为假)
* 合并文件的大小:hive.merge.size.per.task=25610001000(默认值256000000)
### group by 替代 count(distinct)达到优化效果
计算uv的时候，经常会用到count(distinct)，但在数据比较倾斜的时候，count(distinct)会比较慢。可以尝试用group by 改写代码计算uv。

原有代码
```sql
insert overwrite table s_dw_tanx_adzone_uv partition(ds=20120329)
  select 20120329 as thedate,
         adzoneid,
         count(distinct acookie) as uv
  from s_ods_log_tanx_pv t
  where t.ds=20120329 group by adzoneid    
```

### 总结
使map的输出数据更均匀的分布到reduce中去，是最终目标。由于Hash算法的局限性，按key hash会或多或少造成数据倾斜。通用步骤。
* 采样log表，哪些user_id比较倾斜，得到一个结果表tmp1。由于对计算框架来说，所有的数据过来，他都是不知道数据分布情况的，所以采样必不可少
* 数据的分布符合社会学统计规律，倾斜的key不会太多，所以tmp1记录数会很少。把tmp1和users做map join生成tmp2，把tmp2读到distribute file cache。这是一个map过程
* map读入users和log,假如记录来自log，则检查user_id是否在tmp2里，如果是，输出到本地文件a，否则生成的key,value对，假如记录来自membet，生成的key,value对，进入reduce阶段
* 最终把a文件，把Stage3阶段输出的文件合并起写到hdfs
优化方案：
1. 对于join，在判断小表不大于1G的情况下，使用map join
2. 对于group by 或distinct 设定hive.groupby.skewindata=true
3. 尽量使用上述的sql语句调节进行优化
