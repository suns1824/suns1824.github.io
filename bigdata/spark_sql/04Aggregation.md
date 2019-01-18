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
#### DeclarativeAggregate聚合函数
#### InperativeAggregate聚合函数

ps: spark sql学习和之后的改进后续直接在SparkPro工程的rasuf包中进行。