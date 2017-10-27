---
title: hadoop
tags:
 - hadoop
categories: 经验分享
---
### Spark SQL总体设计
前身是Shark，由于Shark对于Hive的太多依赖制约了Spark的发展，Spark SQL由此产生。
```
select name from people where age>=13 and age<=19
```
这条SQL语句由Projection(name)、Data Source(people)、Filter(age>=13 AND age<=19)组成，分别对应于SQL查询过程中的Result、Data Soource、Operation，即SQL语句是按照Result->Data Source->Operation的次序来描述的

#### 传统关系型数据库SQL运行原理
先将SQL语句(Query)进行解析(Parse),找出其中的关键词(如SELECT、FROM、WHERE)、表达式、Projection(a1,a2,a3)、Data Source(tableA) 接着会对SQL语句校验规范，如果规范则将SQL语句和数据库的数据字典(列、表、视图等)进行绑定(Bind)，
在执行前，数据库会从多个执行计划中选择一个最优计划(Optimize),最后执行该计划(Execute)，并返回结果。整个过程与SQL语句次序正好相反

Query->Parse->Bind->Optimize->Execute
#### Spark SQL运行架构
基于Spark SQL 可以重用Hive本身提供的元数据仓库(MetaStore)、HiveQL、用户自定义函数(UDF)及序列化和反序列化的工具(SerDes)

SQLContext由以下部分组成：
* Catalog:字典表，用于注册表，对表缓存后便于查询
* DDLParser:用于解析DDL语句，如创建表
* SparkSQLParser:作为SqlParser的代理，处理一些SQL中的通用关键字
* SqlParser:用于解析select查询语句
* Analyzer:对还未未分析的逻辑执行计划(LogicalPlan)进行分析
* Optimizer:对已经分析过得逻辑执行计划(LogicalPlan)进行优化
* SparkPlanner:用于将逻辑执行计划(LogicalPlan)转换为物理执行计划(PhysicalPlan)
* prepareForExecution:用于将物理执行计划(PhysicalPlan)转换为可执行物理计划
这些部分共同参与到SQL的执行过程中，步骤如下：
1. SQL语句经过SqlParser解析成Unresolved LogicalPlan
2. 使用Analyzer结合数据字典(catalog)进行绑定，生成Resolved LogicalPlan
3. 使用Optimizer对Resolved LogicalPlan进行优化，生成Optimized LogicalPlan
4. 使用SparkPlan将LogicalPlan转换成PhysicalPlan
5. 使用prepareForExecution将PhysicalPlan转换成可执行物理计划
6. 使用execute()执行可执行物理计划，生成SchemaRDD

×->sqlText->SqlParser->UnresolvedLogicalPlan->Analyzer->resolvedLogicalPlan->Optimizer->optimizedLogicalPlan->SparkPlan->PhysicalPlan->prepareForExecution->可执行PhysicalPlan->execute->SchemaRDD->X

### 字典表Catalog
Catalog是一个特质，定义了以下接口：
* tableExists:判断表是否存在
* lookupRelation:使用表名查找关系
* registerTable:注册表
* unregisterTable:取消注册表
* unregisterAllTables:消除所有已经注册的表
### Tree和TreeNode
Spark的语法树是由TreeNode实现的

除了Scala中耳熟能详的方法foreach、map、flatMap、collect外，TreeNode还实现了应用Rule的方法

TreeNode共有3种：
* UnaryNode:一元节点，即只有一个子节点的节点。如Project、Sort、Limit、Filter等操作
* BinaryNode:二元节点，即有左右两个节点的节点。如Except、Intersect操作
* LeafNode:叶子节点，即没有子节点的节点。如
