Join将多个关系数据表按照一定的条件连接在一起。分布式环境下，因为涉及数据在不同节点间Shuffle，Join可以算是代价最昂贵的操作。
### Join查询概述
```text
select name, score from student join exam where student.id = exam.studentId
```
#### 文法定义与抽象语法树
理解该查询所生成的AST!
#### Join查询逻辑计划
与Join算子相关的部分在From子句中，逻辑计划生成过程由AstBuilder类定义的visistFromClause方法开始：
```text
override def visitFromClause(ctx: FromClauseContext): LogicalPlan = withOrigin(ctx) {
  //ctx.relation得到的是RelationContext列表，其中有一个RelationPrimaryContext(主要的数据表)和多个需要Join连接的表
  //
  val from = ctx.relation.asScala.foldLeft(null: LogicalPLan) { (left, relation) =>
    val right = plan(relation.relationPrimary)
    val join = right.optionalMap(left)(Join(_, _, Inner, None))
    withJoinRelations(join, relation)
  }
  ctx.lateralView.asScala.foldLeft(from)(withGenerate)
}
```

#### 从Unresolve LogicalPlan到Analyzed Logical Plan
ResolvRelations和ResolveReferences两条规则产生了影响。ResolveRelations规则从Catalog中找到student和exam的基本信息，包括数据表存储格式，每一列列名和数据类型等。  
ResolveReferences规则负责解析所有列信息，对于上面的逻辑算子树，ResolveReferences的解析是一个自底向上的过程，将所有的UnresolvedAttribute和UnresolvedExtractValue类型的表达式解析成对应的列信息。   
**ResolveReferences中处理Join中冲突的列的逻辑：**
```text
// 针对存在冲突的表达式会创建一个新的逻辑计划，通过增加别名的方式来避免列属性的冲突，将右子节点对应的expression用一个新的Expression ID表示，因此可以区分Join操作中不同的数据表。
private def dedupRight (left: LogicalPlan, right: LogicalPlan): LogicalPLan = {
  val conflictingAttributes = left.outputSet.intersect(right.outputSet)
  ......
}
case j @ Join(left, right, _, _) if !j.duplicateResolved =>
  j.copy(right = dedupRight(left, right))

```
在Analyzer中，还有一个和Join操作直接相关的ResolveNaturalAndUsingJoin规则：将NATURAL或USING类型的Join转换为普通Join。   
#### 从Analyzed LogicalPlan到Optimized LogicalPlan
1. 消除多余的别名 -- EliminateSubqueryAliases优化规则。  
2. 列剪裁 -- Column Pruning。  
3. 优化过程将考虑相关算子中的过滤条件。对于Join来讲，其连接条件需要保证两边的列都不为null，因此会触发InferFiltersFromConstraints规则(本例中，Join算子的裂解条件多了两个，分别约束student表中
的ID和exam表中的studentID不为null)。  
4. 在Optimizer阶段，有一条专门针对Join算子的PushPredicateThroughJoin优化规则：对Join中连接条件可以下推到子节点的谓词进行下推操作。  
5. 经过PushPredicateThroughJoin优化规则后，Join中的两个连接条件生成了对应的两个Filter节点。一般来讲，优化阶段会将过滤条件尽可能下推，因此逻辑算子树中的Filter节点还会被继续处理。该逻辑对应
PushDownPredicate优化规则。  

### Join查询物理计划
#### Join物理计划的生成
从逻辑计划到物理计划的生成是基于策略进行的。本例中逻辑算子树将会使用3个策略： 文件数据源策略(FileSource),Join选择(JoinSelection)策略和基本算子(BasicOperators)策略。JoinSelection策略主要根据
Join逻辑算子选择对应的物理算子。 
#### Join物理计划的选取
在生成物理计划的过程中，JoinSelection根据若干条件判断采用何种类型的Join执行方式。目前在Spark SQL中有5种Join的执行方式：
>* BroadcastHashJoinExec
>* ShuffledHashJoinExec
>* SortMergeJoinExec
>* BroadcastNestedLoopJoinExec
>* CartesianProductExec

**ps: 理解JoinSelection的执行流程！！**

### Join查询执行
#### Join执行基本框架
参与Join操作的两张表分别被称为“流式表”和“构建表”，不同的角色在Spark SQL中会通过一定的策略进行设定。通常来讲，系统默认将大表设定为流式表，小表设定为构建表。流式表的迭代器为streamedIter，构建表
的迭代器为buildIter。遍历streamIter中的每条记录，然后在buildIter中查找相匹配的记录。这个查找过程称为Build过程。每次Build操作是结果为一条JoinedRow(A, B),其中A来自streamIter，B来自buildIter，这个
过程称为BuildRight操作，反之XXX。   
在具体的Join实现层面，Spark SQL提供了BroadcastJoinExec，ShuffleHashJoinExec和SortMergeJoinExec这3种机制。
#### BroadcastJoinExec机制
该Join实现的主要思想是对小表进行广播操作，避免大量shuffle的产生(参考事实表与维度表的设计)。  
ps：需要注意的是，在Outer类型的Join中，基表不能被广播。BroadcastJoinExec只适合用于广播较小的表，否则数据的冗余传输远大于Shuffle的开销。另外，广播时需要将被广播的表读取到Driver端，当频繁有广播出现时，
对Driver端的内存会造成压力。   
#### ShuffledHashJoinExec执行机制
理解其满足条件，执行分为两步：  
1. 对两张表分别按照Join key重分区，即shuffle,目的是为了让有相同key值的记录分到对应的分区中，这一步对应执行计划中的Exchange节点 。  
2. 对每个对应分区中的数据进行Join操作，此处先将小表分区构造成一张hash表，然后根据达标分区中记录的key值进行匹配，即执行计划中的ShuffledHashJoinExec节点。   
其父类HashJoin的要点：
>* boundCondition和连接条件属性condition等价，用于判断一行数据是否满足Join条件
>* (buildPlan, streamedPlan)与(buildKeys, streamedKeys)用于分区两个表的角色(流式表，构建表)
>* 构建表在join过程中会创建一个HashMap，用来支持数据查找，而流式表一行行地在构建表对应的Hashmap中查找数据  
>* Stream表属于动态的一方，会涉及数据的分区，因此其outputPartitioning由StreamPlan中的输出分区决定  
>* output函数表示输出的列
>* Join函数的输入参数包含一个HashedRelation类型，对应构建表，起到了HashMap的作用

理解了HashJoin，再来理解ShuffledHashJoinExec就比较简单了。
```text
protected override def doExecute(): RDD[InternalRow] = {
  //对构建表建立HashedRelation,然后调用HashJoin中得到join方法，对当前流式表中的数据行在HashedRelation中查找数据进行Join操作
  streamedPlan.execute().zipPartitions(buildPlan.execute()) { (streamIter, buildIter) =>
    val hashed = buildHashedRelation(buildIter)
    join(streamIter, hashed, numOutputRows)
  }
}
```

#### SortMergeJoinExec执行机制
Hash系列的Join实现中都是将一侧的数据完全加载到内存中，当两个数据表都很大的时候，无论使用哪种方法都会对计算内存造成很大压力，这时，Spark SQL采用SortMergeJoinExec的方法。
**SortMergeJoinExec并不将一侧数据全部加载后再进行Join操作，其前提条件是需要在Join操作前将数据排序**    
一切分析参考SortMergeJoinScanner类。  


