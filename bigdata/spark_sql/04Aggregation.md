**聚合操作(Aggregation)是指在原始数据的基础上按照一定的逻辑进行整合，从而得到新的数据，一般通过聚合函数汇总多行的信息。**
## Aggregation执行概述
在Catalyst的SqlBase.g4文法文件中，聚合语句aggregation定义如下： 在常见的聚合查询中，通常包括分组语句(group by)和聚合函数(aggregate function)。除了简单的分组操作外，聚合查询海之此后OLAP常见下的多维分析，
包括rollup，cube，groupingsets3种操作。   
如下实例：
```text
select id, count(name) from student group by id
```
首先先得出聚合查询抽象语法树，相对于之前例子的AST，多了FunctionCallContext节点和Aggregation节点。  
#### Unresolved LogicalPlan生成
#### 从逻辑算子树到物理算子树
PhysicalAggregation策略在提取信息时会进行一下转换：
>* 去重：多次出现的聚合操作进行去重
>* 命名
>* 分离

### 聚合函数
#### 聚合缓冲区
#### 聚合模式
>* Partial模式
>* PartialMerge模式
>* Final模式
>* Complete模式

Partial模式和Final模式一般组合在一起使用，对应Map和Reduce阶段。Complete模式的最终阶段直接针对原始数据。PartialMerge模式 聚合函数主要时对聚合缓冲区进行合并。   

### 聚合函数
聚合函数是聚合查询中非常重要的元素。   
实现上，聚合函数是表达式中的一种，和Catalyst中定义的聚合表达式(AggregationExpression)紧密关联，无论在逻辑算子树还是在物理算子树中，聚合函数都是以聚合表达式的形式封装的。
介绍三种聚合函数：
>* DeclarativeAggretate聚合函数，在Count类中有所展示
>* ImperativeAggregate聚合函数
>* TypedImperativeAggregate聚合函数

### 聚合执行
本质上是将RDD的每个Partition中的数据进行处理。2.1中，聚合查询的最终执行方式有两种：SortAggregateExec和HashAggregateExec。后续有新增。
什么时候使用sortAggregation？
>* 查询中存在不支持Partial方式的聚合函数：此时会调用AppUtils中的planAggregateWithoutPartial方法，直接生成SortAggregateExec聚合算子节点。
>* 聚合函数结果不支持Buffer方式：如果结果类型不属于[NullType, BooleanType......]中的一种，需要执行SortAggregateExec方式，例如：collect_set和collect_list函数。
>* 内存不足

聚合查询的执行过程中有一个通用的框架，主要接口定义在AggregationIterator中。
聚合执行框架指的是聚合过程中抽象出来的通用功能，包括聚合函数的初始化，聚合缓冲区更新合并函数以及聚合结果生成函数等。  
AggtrgationIterator中定义了3个主要功能(两个实现类，分别对应SortAggregationExec和HashAggregationExec两种执行方式)：
>* 聚合函数初始化
>* 数据处理函数生成
>* 聚合结果输出函数生成  

#### 基于排序的聚合算子SortAggregateExec
#### 基于Hash的聚合算子HashAggregateExec 

### 窗口函数
窗口函数和普通聚合函数类似，都是对多行数据进行整合，不同之处在于窗口函数多了一个灵活的“窗口”，支持用户指定更加复杂的聚合行为(如数据划分和范围设置)。    
窗口函数涉及3个核心元素：
 >* 分区信息(partitionBy子句定义，类似SparkPlan中的Partitioning，数据基于分区表达式执行Hash类型的Shuffle操作)
 >* 排序信息(定义了分区内数据的顺序)
 >* 窗框定义(窗框是一个在分区内对行进行进一步限制的筛选器)  
 
 ps：理解如下SQL语句的大致执行流程：
 ```text
 // 根据gradeId，classID分区，然后执行orderby，最后......
select studentID, row_number() over (partition by gradeID, classID order by score desc) as ranking from exam
```
总体来看，窗口函数除了输入，输出行相等外，还包括如下特性和优势：
>* 类似Group By的聚合，支持非顺序的数据访问  
>* 可以对窗口函数使用分析函数，聚合函数和排名函数
>* 简化了SQL代码(消除Join)并可以避免中间表  

#### 窗口函数相关表达式
在Catalyst中，窗口表达式(WindowExpression)包含了WindowExpression和WindowSpecDefinition类型(包含了3个核心元素)。
#### 窗口函数的逻辑计划阶段与物理计划阶段
#### 窗口函数的执行 

### 多维分析
#### OLAP多维分析
OLAP多维分析支持用户从多个角度和多个侧面去考察数据仓库中的数据。理解如下概念：
>* 维：人们观察数据的特定角度,比如日期维度[维度表](https://www.jianshu.com/p/51ddd3967709)  
>* 维的层次：人们观察数据的特定角度(某个维)还可以存在细节，程度等多个方面的描述
>* 维成员：维的一个取值称为该维的一个维成员。如果一个维是多层次的，那么该维的维成员是在不同维层次的取值组合
>* [多维数据集](https://blog.csdn.net/u013323965/article/details/73929009)： 多维数据集是决策支持的支柱，也称立方体
>* [数据单元](https://docs.microsoft.com/zh-cn/sql/analysis-services/multidimensional-models-olap-logical-cube-objects/cube-cells-analysis-services-multidimensional-data?view=sql-server-2017)： 多维数据集的取值称为数据单元
>* 多维数据集的度量值： 在多维数据集中有一组度量值，这些值基于多维数据集中事务表的一列或多列

为了保证信息处理所需的数据以合适的粒度，合理的抽象程度和标准化程度存储，数据在物理上分为3种存储结构：基于多维数据库的OLAP，基于关系数据库的OLAP，混合型的OLAP(理解三者特点)。  
#### Spark SQL多维查询
SQL文法种多维分析的关键字有cube，rollup，grouping sets3种，举个例子：
```text
//group by子句中指定了维度列和关键字with cube，得理解查询结果。
select gradeId, classId, max(score) from exam group by gradeId, classId with cube
```


