---
title: hadoop
tags:
 - hadoop
categories: 经验分享
---

DataSet是特定域对象中的强类型集合，它可以使用函数或者相关操作并行地进行转换等操作。每个DataSet都有一个称为DataFrame的非类型化的视图，这个视图是行的数据集。

RDD也是可以并行化的操作，DataSet和RDD主要的区别是：DataSet是特定域的对象集合;然而RDD是任何对象的集合。DataSet的API是强类型化的。而且可以利用这些模式进行优化，然而RDD却不行

DataFrame是特殊的DataSet，它在编译时不会对模式进行检测。

### DataSet Wordcount实例
1. 创建SparkSession
```
val sparkSession = SparkSession.builder.
      master("local")
      .appName("example")
      .getOrCreate()
```
2. 读取数据并将它转换成DataSet

read.text如RDD版提供的textFile，as[String]可以为dataset提供相关的模式
```
import sparkSession.implicits._
val data = sparkSession.read.text("src/main/resources/data.txt").as[String]
```
3. 分割单词并且对单词进行分组
DataSet提供的API和RDD提供的非常类似，所以可以在DataSet对象上使用map，groupByKey相关API
```
val words = data.flatMap(value => value.split("\\s+"))
val groupedWords = words.groupByKey(_.toLowerCase)
```
DataSet是工作在行级别的抽象，每个值被看作是带有多列的行数据，而且每个值都可以看作是group的key，正如关系型数据库的group
4. 计数
如RDD上reduceByKey
```
val counts = groupedWords.count()
```
5. 打印结果
```
counts.show()
```
